# Tech Stack

## Runtime
- **Language**: Go (both repos)
- **Web framework**: Gin (3rd-payment only)
- **Process model**:
  - 3rd-payment: long-running HTTP server (port 8081) + separate `StartCronjob()` entry
  - que_payment: long-running worker, switches behavior via `TYPE` and `PAYMENT_NAME` env

## Storage
- **MongoDB** — primary store (orders, statements, config, bank summary, ...)
- **Redis** — distributed lock (thorlock), cache, รับ event จาก cronjob
- **RabbitMQ** — work queue, 1 queue / provider (`QUE_PAYMENT_<NAME>`), prefetch=1, manual ack
- **ClickHouse** — event/error log table `payment_events` (MergeTree, partition by month, batch 1s/1000 rows)

## Observability
- **OpenTelemetry** via `github.com/Maximumsoft-Co-LTD/otelgo/eto`
  - HTTP middleware: W3C trace propagation + root span per request
  - AMQP: context propagation through message headers (`amqpcarrier.AmqpHeadersCarrier`)
- **Prometheus** — `metrics.NewPrometheus("gin")` (3rd-payment only)
- **Telegram alerting** — 2 chats: regular + on-call
  - severity 2-3 → regular chat, rate-limited 1 msg/s, dedup 5min
  - severity 4 → on-call chat (no rate-limit)
  - dedup key: `hash(severity + payment_code + error_type)`
- **Grafana Tempo** — trace links embedded in Telegram alerts (env `GRAFANA_TEMPO_URL`)

## External integrations
- ~60 payment provider APIs (each as its own folder)
- Telegram Bot API (both business bot และ error bot)
- Optional: KBiz, SCB, BBL endpoints (ดูใน `bank-gateway/*` routes)

## Env vars สำคัญ (จาก app.go และ README)
```
PROJECT                                  # service name (used in traces, alerts)
PAYMENT_NAME                             # ใช้ใน que_payment, ระบุว่าเป็น provider ไหน
TYPE                                     # QUE_RABBIT | QUE_CRONJOB (que_payment)
RABBIT_MQ / RABBIT_MQ_DEVELOPMENT        # AMQP URL
REDIS_URL / REDIS_DB
OTEL_CONNECT                             # OTLP gRPC endpoint
GRAFANA_TEMPO_URL
TELEGRAM_ERROR_BOT_TOKEN
TELEGRAM_ERROR_CHAT_ID
TELEGRAM_ERROR_ONCALL_CHAT_ID
CLICKHOUSE_DSN
BOT_TELEGRAM_TOKEN                       # 3rd-payment business bot
BOT_TELEGRAM_URL_WEBHOOK
BOT_TELEGRAM_SECRET
MODE                                     # production | (other)
```

## Distributed lock config
- `thorlock.InitLock(time.Second*180, time.Second*180, time.Second*10, 18)` — wait/lease 180s, retry 10s, DB 18
