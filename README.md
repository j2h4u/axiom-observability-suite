# axiom-observability-suite

Готовый стек мониторинга для Docker-серверов. Ставится за минуту, работает незаметно,
пишет в Telegram когда что-то идёт не так.

Логи хранятся в [Axiom](https://axiom.co) — облачная платформа для хранения и анализа логов
(бесплатный план включён).

## Как устроен

Пакет делает две независимые вещи. Можно использовать обе или только первую.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Функция 1: СБОР                       ставится на каждый сервер   │
│                                                                     │
│  Docker контейнеры                                                  │
│    ├─ stdout/stderr ──→ Vector ──→ Axiom (облачное хранилище логов) │
│    └─ healthcheck ────→ health-watcher ──→ алерт при unhealthy      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Функция 2: ДОСТАВКА АЛЕРТОВ            ставится на один сервер    │
│                                                                     │
│  Axiom Monitor ──webhook──→ alertbot ──routes.yml──→ Telegram       │
│                                                       (топики,      │
│                                                        приоритеты,  │
│                                                        runbook)     │
└─────────────────────────────────────────────────────────────────────┘
```

### Функция 1: сбор логов и health-статусов

Ставится на каждый сервер, где есть Docker. Один `docker compose up -d` — и все
контейнеры на хосте под наблюдением. Ничего не нужно менять в коде или конфигах
самих сервисов.

- **axiom-log-shipper** (Vector) — читает stdout всех контейнеров и отправляет в Axiom.
  Логи доступны в облаке для поиска и анализа.
- **axiom-health-watcher** — слушает Docker events. Если healthcheck контейнера
  переходит в `unhealthy` и остаётся там дольше двух минут — алерт.

### Функция 2: адаптер Axiom → Telegram

Ставится только на одном сервере (профиль `alertbot`). Это переходник: принимает
webhook от Axiom Monitor, форматирует в читаемое сообщение и доставляет в нужный
топик Telegram-группы.

- **axiom-to-telegram-bot** — маршрутизация по `routes.yml` (какой сервис → какой топик),
  приоритеты, runbook-подсказки, дедупликация.

Axiom анализирует поток логов. Когда монитор видит ошибки — шлёт webhook.
Alertbot превращает его в понятное Telegram-сообщение. Один алерт за пять минут
максимум — без шквала одинаковых.

### Self-monitoring

Сам стек мониторинга тоже под наблюдением. Если падает бот или Vector,
health-watcher доставит алерт напрямую в Telegram, минуя сломанный компонент.
Петли и амплификация исключены архитектурно — подробности в [docs/ALERTING.md](docs/ALERTING.md).

## Что нужно

- Docker Engine 20+ и docker compose v2
- Аккаунт [Axiom](https://axiom.co) (бесплатный план достаточен)
- Для alertbot: Telegram-бот, домен с HTTPS

## Быстрый старт

```bash
cp .env.example .env
# Заполнить AXIOM_TOKEN, AXIOM_DATASET, HOSTNAME
docker compose up -d
```

Через минуту проверь, что логи дошли: в Axiom открой датасет → Stream.
Должны появиться события. Если пусто — `docker logs axiom-log-shipper --tail 20`.

### С alertbot (на одном сервере)

```bash
# Дополнительно в .env:
# COMPOSE_PROFILES=alertbot
# TELEGRAM_BOT_TOKEN, AXIOM_MGMT_TOKEN
# + скопировать routes.yml из routes.yml.example (маршрутизация алертов по топикам)
docker compose --profile alertbot up -d
```

Нужен reverse proxy (nginx или Caddy) для публичного URL вебхука — см. [docs/ALERTING.md](docs/ALERTING.md).

## Документация

- [docs/GETTING-STARTED.md](docs/GETTING-STARTED.md) — сценарии подключения, перенос на другой сервер
- [docs/SERVICE-GUIDE.md](docs/SERVICE-GUIDE.md) — чеклист для сервиса: healthcheck, логирование, Vector
- [docs/AXIOM.md](docs/AXIOM.md) — APL-запросы, мониторы, CLI (`axiom_cli.py`), MCP
- [docs/ALERTING.md](docs/ALERTING.md) — alertbot, маршрутизация, self-monitoring, лучшие практики алертов

## Управление

```bash
# Статус
docker ps | grep axiom-

# Логи
docker logs axiom-log-shipper --tail 50
docker logs axiom-health-watcher --tail 50
docker logs axiom-to-telegram-bot --tail 50

# Перезапуск (новый .env)
docker compose --profile alertbot up -d
```
