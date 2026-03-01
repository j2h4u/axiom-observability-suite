[← README](../README.md)

# Axiom: запросы, мониторы, CLI

Кому читать: DevOps и операторам стека.

Датасет: `prod-docker-logs`. Это имя захардкожено в `axiom_cli.py` — мониторы
создаются именно для этого датасета. Если нужен другой — поменять константу `DATASET`
в скрипте. Поля в каждом событии: `host`, `service`, `message`, `_time`.

---

## APL-запросы

**Логи сервиса:**
```
['prod-docker-logs']
| where service == "my-service"
| order by _time desc
| limit 200
```

**Только ошибки:**
```
['prod-docker-logs']
| where service == "my-service"
| where message contains_cs "ERROR" or message contains_cs "Traceback"
| order by _time desc
| limit 200
```

**За последний час:**
```
['prod-docker-logs']
| where _time > ago(1h)
| where service == "my-service"
| order by _time desc
| limit 200
```

Если `service` не совпадает: `| where container_name contains "my-service"`.

---

## axiom_cli.py — управление мониторами

```bash
cd /opt/axiom-observability-suite
python3 axiom_cli.py <команда>
```

Токен читается из `.env` (`AXIOM_MGMT_TOKEN`).

**Мониторы:**
```bash
python3 axiom_cli.py monitors list
python3 axiom_cli.py monitors create <service>                          # порог: 1 ошибка / 5 мин
python3 axiom_cli.py monitors create <service> --interval 10 --threshold 3
python3 axiom_cli.py monitors create-health-watcher                     # монитор для unhealthy-контейнеров
python3 axiom_cli.py monitors attach-notifiers
python3 axiom_cli.py monitors delete <id>
```

`create-health-watcher` создаёт Threshold-монитор, который алертит если health-watcher
зафиксировал unhealthy-контейнер. Нужен один раз на инсталляцию.

**Нотификаторы:**
```bash
python3 axiom_cli.py notifiers list
python3 axiom_cli.py notifiers create <name> <webhook-url>
python3 axiom_cli.py notifiers delete <id>
```

См. [ALERTING.md](ALERTING.md) (раздел "Лучшие практики по алертам Axiom") — формат вебхука и рекомендации по содержанию алертов.

**Обязательная привязка notifier:** Axiom позволяет создать монитор без notifier —
в таком случае алерты не уходят. Регламент: создавать мониторы через
`axiom_cli.py`. После любых изменений мониторов или деплоя безусловно запускать
`python3 axiom_cli.py monitors attach-notifiers` — команда проставляет notifier
всем мониторам, где он отсутствует. Если notifiers пустые — привяжи notifier в UI
или пересоздай монитор через CLI.
Если alertbot запущен с `AXIOM_MGMT_TOKEN`, он выполняет auto-attach каждые 5 минут.

**Дедупликация:** монитор срабатывает максимум раз в `intervalMinutes`.
1000 одинаковых ошибок за 5 минут → один алерт.

**Что ловит монитор** (APL-запрос, который генерирует скрипт):
```
['prod-docker-logs']
| where service == "<service>"
| where message contains "ERROR" or message contains "error"
   or message contains "Traceback" or message contains "Exception"
| count
```

**Токен для CLI:** `AXIOM_MGMT_TOKEN` в `.env`.
Права: Monitors (CRU), Notifiers (CRU), Datasets (R), Queries (R).
Создать: Axiom → Settings → API Tokens → New Token → Advanced.

---

## Текущее состояние

После деплоя проверь текущие ресурсы:
```bash
python3 axiom_cli.py notifiers list
python3 axiom_cli.py monitors attach-notifiers
python3 axiom_cli.py monitors list
```

---

## Подключение MCP (для AI-клиентов)

Axiom поддерживает MCP через OAuth — работает в Claude Code, Cursor и других клиентах.

1. Добавь remote MCP server: `https://mcp.axiom.co/mcp`
2. Авторизуйся в браузере через Axiom

Если OAuth не работает (headless-сервер), используй PAT:
```
Authorization: Bearer <PAT>
x-axiom-org-id: <ORG_ID>
```

MCP поддерживает: `queryApl`, `listDatasets`, `getDatasetSchema`, `getMonitors`, `getMonitorsHistory`.
Создавать и редактировать мониторы через MCP нельзя — используй `axiom_cli.py`.
