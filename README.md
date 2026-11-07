# monitoring-compose

Учебный стенд мониторинга на двух VPS: метрики (Prometheus + Grafana) и логи (ELK), всё в Docker Compose. Grafana и Kibana открыты из интернета по HTTPS с логином через nginx.

## Схема

```
 VPS A (elk-vps)                                VPS B (monitoring-vps)
 ┌───────────────────────────────┐              ┌───────────────────────────────┐
 │ nginx :443 → Kibana           │              │ nginx :443 → Grafana          │
 │ Elasticsearch  (127.0.0.1)    │              │ Prometheus     (127.0.0.1)    │
 │ Logstash :5044 ◀──────────────┼── логи ──────┤ Filebeat                      │
 │ Filebeat → Logstash (локально)│              │ node-exporter                 │
 │ node-exporter :9100 ◀─────────┼── метрики ───┤ Prometheus забирает (scrape)  │
 └───────────────────────────────┘              └───────────────────────────────┘
 kibana.artemdevops.ru                          grafana.artemdevops.ru
```

- **Метрики:** Prometheus на B сам опрашивает node-exporter на обоих серверах; Grafana рисует графики (дашборд Node Exporter Full, номер 1860).
- **Логи:** Filebeat на каждом сервере читает системный журнал (journald) и отправляет в Logstash на A, оттуда в Elasticsearch; смотрим в Kibana.
- Без SSH-туннелей: логи идут прямо на порт 5044 сервера A, метрики A забирает B с порта 9100. Оба порта в ufw открыты только для IP соседнего сервера.

## Структура

| Путь | Что там |
|---|---|
| `elk/` | Compose для VPS A: Elasticsearch, Kibana, Logstash, Filebeat, node-exporter |
| `elk/logstash/pipeline/` | pipeline Logstash (приём от Filebeat, запись в Elasticsearch) |
| `elk/filebeat/` | конфиг Filebeat |
| `elk/nginx/` | конфиг nginx для Kibana |
| `prometheus-grafana/` | Compose для VPS B: Prometheus, Grafana, node-exporter, Filebeat |
| `prometheus-grafana/prometheus/` | конфиг Prometheus и список серверов для опроса (`targets/node.yml`) |
| `prometheus-grafana/grafana/provisioning/` | источник данных Grafana, подключается файлом при старте |
| `prometheus-grafana/filebeat/` | конфиг Filebeat |
| `prometheus-grafana/nginx/` | конфиг nginx для Grafana |

## Как устроено и почему

- Имя проекта (`name:`) и версии образов заданы явно; версии и пароли берутся из файла `.env` (в репозитории его нет, он в `.gitignore`).
- Порты контейнеров опубликованы только на `127.0.0.1`; снаружи входит nginx на 80/443. Docker обходит ufw для опубликованных портов, поэтому защита сделана привязкой к loopback.
- Исключения, где используется `network_mode: host`: Logstash (принимает логи с другого сервера на 5044), Filebeat на A, node-exporter (должен видеть сам сервер). В host-режиме действует ufw.
- Prometheus на B достаёт node-exporter своего сервера по адресу шлюза Docker-сети (`172.30.0.1:9100`), для этого в ufw есть правило для подсети `172.30.0.0/24`.
- Доступ к Kibana и Grafana: собственные формы входа; для просмотра создан отдельный пользователь с правами только на чтение.

## Переменные `.env`

Файл `.env` создаётся на сервере вручную, значения не коммитятся.

**VPS A (`elk/.env`):** `ELASTIC_VERSION`, `NODE_EXPORTER_VERSION`, `KIBANA_DOMAIN`, `ELASTIC_PASSWORD`, `KIBANA_PASSWORD`, `LOGSTASH_INTERNAL_PASSWORD`, `KIBANA_ENCRYPTION_KEY`.

**VPS B (`prometheus-grafana/.env`):** `PROMETHEUS_VERSION`, `GRAFANA_VERSION`, `NODE_EXPORTER_VERSION`, `ELASTIC_VERSION`, `GRAFANA_DOMAIN`, `GRAFANA_ADMIN_PASSWORD`, `HOST_NAME`, `LOGSTASH_HOST`.

## Порядок запуска (кратко)

1. Подготовить сервер: обновления, swap, Docker, ufw, DNS для поддоменов.
2. На A: поднять Elasticsearch; создать пользователя и роль для Logstash, задать пароль `kibana_system`; один раз выполнить `filebeat setup --index-management` (создаёт data stream); затем поднять остальное.
3. На B: `docker compose up -d`.
4. На обоих: nginx + `certbot --nginx` для выпуска сертификата.
5. Проверить: цели Prometheus в состоянии up, в Kibana (`filebeat-*`) поле `host.name` содержит оба сервера.

Подробная пошаговая инструкция будет добавлена в `docs/`.

## Известные ограничения

- Логи и метрики между серверами идут по интернету без шифрования; защита только файрволом по IP.
- Внутри сети Docker TLS отключён (учебное упрощение).
- Нет политики хранения логов (ILM), защищённой очереди Logstash и алертов (Alertmanager).
