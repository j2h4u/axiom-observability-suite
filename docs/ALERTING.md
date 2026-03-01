[← README](../README.md)

# Alerting: Axiom → Telegram

Кому читать: тем, кто настраивает алерты, Telegram и маршрутизацию по топикам.

---

## Архитектура

```
Axiom Monitor   → POST /webhook/axiom  ─┐
                                         ├→  alertbot  ──routes.yml──→  Telegram topics
health-watcher  → POST /alert/local    ─┘
```

- `/webhook/axiom` — внешний путь для Axiom Monitors (через reverse proxy)
- `/alert/local` — внутренний путь для health-watcher (Docker-сеть, без auth)

Подробнее о self-monitoring — см. раздел ниже.

---

## Расположение

```
/opt/axiom-observability-suite/
├── app.py              # весь код axiom-to-telegram-bot
├── Dockerfile
├── docker-compose.yml  # axiom-log-shipper + axiom-to-telegram-bot (profiles)
├── requirements.txt
├── vector.toml         # конфиг axiom-log-shipper
├── routes.yml          # маршрутизация алертов по топикам (не в git)
├── routes.yml.example  # шаблон для routes.yml
├── .env
└── .env.example
```

---

## Конфигурация

### .env

```
TELEGRAM_BOT_TOKEN=     # токен бота от @BotFather
WEBHOOK_SECRET=         # опционально: Axiom передаёт в заголовке X-Webhook-Secret

# Авто-привязка notifiers к мониторам (Axiom)
AXIOM_MGMT_TOKEN=       # PAT, права: Monitors/Notifiers CRU, Datasets/Queries R
AXIOM_ATTACH_INTERVAL_SECONDS=300  # опционально, по умолчанию 300
AXIOM_QUERY_BASE=https://cloud.axiom.co  # база для Axiom-запросов (enrichment)

# Фильтрация шума
ALERTBOT_INCLUDE_RESOLVED=false  # отправлять resolved-события (по умолчанию нет)

# Резерв (если нет routes.yml, а также для health-watcher.sh):
# TELEGRAM_CHAT_ID=
# TELEGRAM_TOPIC_ID=
```

Важно: для enrichment alertbot использует `AXIOM_DATASET` из `.env` (тот же, что у log shipper).

---

### routes.yml (маршрутизация по топикам)

См. `routes.yml.example` — шаблон с комментариями.

Как получить ID топика: из ссылки на сообщение в топике.
`https://t.me/c/3779402801/97/165` → topic_id = `97`, chat_id = `-1003779402801`.

Без `routes.yml` бот fallback'ится на `TELEGRAM_CHAT_ID` / `TELEGRAM_TOPIC_ID` из `.env`.

Если задан `AXIOM_MGMT_TOKEN`, alertbot раз в 5 минут проверяет мониторы в Axiom
и автоматически привязывает webhook notifier к тем, где он отсутствует.
По умолчанию alertbot не отправляет закрывающие (resolved) сообщения, чтобы не шуметь.

---

### Правила routes.yml (кратко)

Цель — низкий шум и понятные, actionable сообщения.

- Если `routes.yml` не проходит схему или плейсхолдеры — сервис должен упасть
- Разрешенные плейсхолдеры в runbook: `{host}`, `{service}`, `{container}`, `{monitor}`
- Шумовые события удаляются полностью (`drop`)
- P1 определяется по профилям (`profiles`)
- Профили назначаются явно в `services`

Минимальная схема YAML:
- `groups`, `topics`, `routes`, `default_group`, `default_topic`
- `tags`, `defaults`, `drop`, `profiles`, `services`

Как интерпретировать алерт:
1) Применить `drop` фильтры — если совпало, алерт не отправлять
2) Найти профили сервиса → если совпало с `p1`, тег `#UserImpact`
3) Иначе тег `#ServiceErrors`
4) Собрать сообщение по дефолтам (count/окно/top error/sample/runbook)

Пример runbook (общий):
- "Зайди по SSH на {host}"
- "Проверь логи контейнера {service}: docker logs {service} --tail 200"

---

## Эндпоинты

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/health` | healthcheck (используется Docker healthcheck) |
| POST | `/webhook/axiom` | вебхуки от Axiom Monitor (внешний, через reverse proxy) |
| POST | `/alert/local` | алерты от health-watcher (внутренний, Docker-сеть) |

Публичный URL (через reverse proxy): `https://your-server.example.com/alertbot/webhook/axiom`

`/alert/local` принимает `{"title": "...", "body": "..."}`, возвращает 502 если Telegram не ответил.

---

## Формат вебхука от Axiom

Alertbot читает из payload:

- `name` — название монитора
- `description` — описание монитора
- `matchedCount` — количество событий
- `queryStartTime` / `queryEndTime` — временной диапазон
- `queryResult.matches[].data.host` — сервер
- `queryResult.matches[].data.service` — сервис/контейнер
- `queryResult.matches[].data.message` — sample сообщений

---

## Управление

```bash
# Статус
docker ps | grep axiom-to-telegram-bot

# Логи
docker logs axiom-to-telegram-bot --tail 30

# Применить новый routes.yml (не перечитывается на лету — нужен рестарт)
docker compose -f /opt/axiom-observability-suite/docker-compose.yml --profile alertbot up -d

# Пересобрать образ (после изменений app.py)
docker compose -f /opt/axiom-observability-suite/docker-compose.yml --profile alertbot up -d --build
```

---

## Настройка Axiom Monitor

В Axiom создать Monitor:

1. **Query** — APL-запрос, например:
   ```
   ['prod-docker-logs']
   | where service == "my-service"
   | where message contains_cs "ERROR" or message contains_cs "Traceback"
   ```
2. **Threshold** — например, `>= 1` за последние 5 минут.
3. **Notifier** — добавить webhook notifier с URL:
   `https://your-server.example.com/alertbot/webhook/axiom`
4. Опционально: добавить кастомный заголовок `X-Webhook-Secret` и прописать его в `.env`.

---

## Self-monitoring: кто сторожит сторожей

Обычные контейнеры мониторятся через цепочку:

```
контейнер → stdout → Vector → Axiom → Monitor → webhook → bot → Telegram
```

Если мониторить observability-стек через эту же цепочку:

- **bot умирает** → webhook приходит в мёртвого бота → тишина
- **Vector умирает** → логи перестают течь → мониторы молчат → тишина
- **Ошибки в боте** → логи ошибок → Axiom → monitor → webhook → бот → ещё ошибки → амплификация

### Решение: отдельный путь

Для `axiom-*` контейнеров health-watcher использует локальный HTTP-эндпоинт бота
вместо Axiom pipeline:

```
Обычные контейнеры:    unhealthy → stdout → Vector → Axiom → Monitor → bot → Telegram
Observability стек:    unhealthy → health-watcher → POST /alert/local → bot → Telegram
```

### Трёхуровневый fallback

Если `axiom-*` контейнер остаётся unhealthy дольше `UNHEALTHY_ALERT_DELAY_SECONDS`:

1. **POST `http://axiom-to-telegram-bot:8080/alert/local`** — основной путь.
   Бот форматирует и шлёт в Telegram. Если Telegram не ответил — возвращает 502.

2. **wget в Telegram API напрямую** — если бот недоступен (connection refused / 502).
   Использует `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `TELEGRAM_TOPIC_ID` из `.env`.

3. **stdout** — последний шанс. На серверах без alertbot это единственный путь,
   события уйдут в Axiom через Vector как обычно.

### Почему это безопасно

- **Нет петли**: `axiom-*` события не идут через Axiom pipeline на серверах с alertbot
- **Нет амплификации**: health-watcher шлёт один алерт на контейнер (проверяет "всё ещё unhealthy?" после задержки)
- **Fallback прозрачен**: если бот лежит — Telegram получит алерт с пометкой "(alertbot unavailable)"

### Healthcheck'и

Все три observability-контейнера имеют Docker healthcheck:

| Контейнер | Проверка | Что подтверждает |
|-----------|----------|-----------------|
| axiom-to-telegram-bot | `python3 urllib → /health` | FastAPI отвечает |
| axiom-log-shipper | `test -f /proc/1/status` | Процесс Vector жив |
| axiom-health-watcher | `docker info` | Docker socket доступен |

health-watcher слушает `health_status` события Docker — когда healthcheck контейнера
переходит в `unhealthy`, он получает событие и обрабатывает его.

### Эндпоинт `/alert/local`

Локальный синк для алертов от внутренних сервисов (health-watcher).

```
POST /alert/local
Content-Type: application/json

{"title": "Container unhealthy: axiom-log-shipper", "body": "Details..."}
```

- Доступен только внутри Docker-сети (не проброшен наружу)
- Не требует аутентификации (внутренний трафик)
- Возвращает `502` если не удалось отправить в Telegram (для fallback)

### Что НЕ делать

- **Не создавать Axiom-мониторы для `axiom-*` сервисов** — это создаст петлю
  (монитор → webhook → бот, который может быть причиной проблемы)
- **Не добавлять heartbeat** — Docker `restart: unless-stopped` обрабатывает крэши,
  а мониторинг "весь VPS лежит" требует внешнего сервиса и выходит за рамки этого стека

---

## Лучшие практики по алертам Axiom

Короткая выжимка по тому, как сделать алерты полезными и малошумными,
с опорой на документацию Axiom и практики SRE.

### Что дает вебхук Axiom

Кастомный webhook настраивается через Go-шаблоны. Ключевые поля:

- Action: Open (triggered) или Closed (resolved)
- Title: название монитора
- Description: описание монитора
- Body: текст (для threshold — описание сработавшего значения)
- Value: значение, вызвавшее алерт (только threshold)
- QueryStartTime / QueryEndTime
- GroupKeys / GroupValues (если в APL есть group by по не-временным полям)
- MatchedEvent (только для Match-мониторов)

Ссылка:
- https://axiom.co/docs/monitor-data/custom-webhook-notifier

### Threshold и Match мониторы

Threshold:
- Хорош для агрегатов (count/rate)
- Вебхук обычно не содержит конкретных строк логов
- Обычно есть только count + окно времени

Match:
- Срабатывает на конкретном событии
- Вебхук содержит MatchedEvent (можно цитировать ошибку)
- Может быть шумным, если не дедуплицировать

### Практики, чтобы алерты были полезными

- Добавлять контекст: host:service
  - Если возможно, группировать по host/service, чтобы иметь GroupKeys/GroupValues
- Показывать 1-3 sample строки ошибки
  - Либо Match-монитор, либо дополнительный запрос в Axiom при триггере
- Резать шум
  - По умолчанию слать только Open/triggered
  - Resolved — только если нужно для процесса
- Делать сообщение коротким и сканируемым
  - Название, host:service, count, окно, короткий sample

### Как это применить в этом стеке

Рекомендуемый формат сообщения:

- Заголовок
- host:service
- count + окно
- top error + 1-2 sample строки
- короткий runbook

Варианты реализации:

1) Оставить Threshold-мониторы и добавить enrichment в alertbot
   - На trigger делать короткий Axiom-запрос и брать sample строки
   - Нужен AXIOM_MGMT_TOKEN и узкое окно времени

2) Добавить Match-мониторы для отдельных сервисов
   - Использовать MatchedEvent из вебхука
   - Threshold оставить для агрегатов

### Общие принципы

- Алерты должны быть actionable и с минимальным шумом
- Лучше алертить симптомы, чем «возможные причины»
- Дедупликация и агрегация обязательны

Ссылки:
- https://sre.google/sre-book/monitoring-distributed-systems/
- https://sre.google/sre-book/practical-alerting/

