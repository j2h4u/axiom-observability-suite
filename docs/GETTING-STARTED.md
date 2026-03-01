[← README](../README.md)

# Начало работы

Кому читать: всем, кто подключает мониторинг к своему серверу или сервису.

Этот документ сценарный. Выбирай свой вариант и следуй шагам.

## Что нужно заранее
- Сервер с Docker и docker compose
- AXIOM_TOKEN (ingest), AXIOM_DATASET, HOSTNAME
- Доступ к папке проекта на сервере (обычно /opt/axiom-observability-suite)

## Сценарий 1: подключить свой бот к общему мониторингу
1. Скопируй проект на сервер (или попроси DevOps).
2. В папке проекта:
   ```bash
   cp .env.example .env
   ```
3. Заполни в `.env`:
   - `AXIOM_TOKEN`
   - `AXIOM_DATASET`
   - `HOSTNAME`
4. Запусти:
   ```bash
   docker compose up -d
   ```
5. Проверь, что контейнеры запущены:
   ```bash
   docker ps | grep axiom-
   ```
6. Убедись, что контейнер твоего бота пишет в stdout.
   - Если `docker logs <container>` показывает логи — все ок.
7. Важно: если у сервиса нет Docker healthcheck, алерты по unhealthy не придут.
   Попроси добавить (пример в [SERVICE-GUIDE.md](SERVICE-GUIDE.md)).
8. Сообщи DevOps имя контейнера твоего сервиса.
   - Они создадут монитор: `python3 axiom_cli.py monitors create <container>`.

## Сценарий 2: свой Telegram-чат и свои алерты
1. Сделай шаги из сценария 1.
2. В `.env` раскомментируй и заполни:
   - `COMPOSE_PROFILES=alertbot`
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_CHAT_ID`
   - `TELEGRAM_TOPIC_ID` (если нужен топик)
   - `AXIOM_MGMT_TOKEN` (если хочешь авто-привязку webhook)
3. Скопируй и настрой `routes.yml`:
   ```bash
   cp routes.yml.example routes.yml
   ```
4. Настрой публичный HTTPS-URL для вебхука.
   - Нужен reverse proxy, путь `/alertbot/webhook/axiom`.
   - Если не хочется разбираться, попроси DevOps.
5. Запусти:
   ```bash
   docker compose --profile alertbot up -d
   ```
6. Создай webhook notifier в Axiom (или попроси DevOps).
   - URL: `https://<domain>/alertbot/webhook/axiom`

## Проверка, что всё работает
- В Axiom в датасете `prod-docker-logs` найди события своего сервиса по полю `service`.
- Убедись, что контейнеры `axiom-log-shipper` и `axiom-health-watcher` в статусе Up.
- Если включен alertbot: попроси DevOps отправить тестовый алерт, чтобы проверить Telegram.

## Кто делает что
- Ты: поднимаешь compose, заполняешь `.env`, даёшь имя контейнера бота.
- DevOps: создаёт монитор и (если нужен alertbot) настраивает webhook и reverse proxy.

## Если что-то не работает
- Проверь, что `AXIOM_TOKEN`, `AXIOM_DATASET`, `HOSTNAME` заполнены.
- Посмотри логи:
  - `docker logs axiom-log-shipper --tail 50`
  - `docker logs axiom-health-watcher --tail 50`
  - `docker logs axiom-to-telegram-bot --tail 50` (если включен alertbot)
- Если непонятно — пришли DevOps вывод этих команд и имя контейнера бота.

## Перенос на другой сервер

1. Скопировать `/opt/axiom-observability-suite/` на новый сервер.
2. Скопировать `.env` (убедиться что `COMPOSE_PROFILES=alertbot` раскомментирован).
3. Скопировать `routes.yml`.
4. Настроить reverse proxy (axiom-to-telegram-bot слушает на `127.0.0.1:8092`):
   - **nginx** (если уже установлен):
     ```nginx
     location /alertbot/ {
         proxy_pass http://127.0.0.1:8092/;
         proxy_set_header Host $host;
     }
     ```
     ```bash
     certbot --nginx -d <domain>
     ```
   - **Caddy** (если nginx нет — проще, TLS из коробки):
     ```
     <domain> {
         reverse_proxy /alertbot/* localhost:8092
     }
     ```
5. `docker compose --profile alertbot up -d --build`
6. Обновить URL вебхука в Axiom Notifier на новый домен.

## Полезные ссылки
- [ALERTING.md](ALERTING.md) — подробности по alertbot и вебхукам
- [SERVICE-GUIDE.md](SERVICE-GUIDE.md) — про healthcheck и логирование
- [AXIOM.md](AXIOM.md) — Axiom, мониторы и CLI
