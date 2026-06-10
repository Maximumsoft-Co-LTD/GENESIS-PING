---
last_reviewed: 2026-06-10
---

# Tech Stack

## Runtime
- **Language**: Go (both repos)
- **Web framework**: Gin (3rd-payment only)
- **Process model**:
  - 3rd-payment: long-running HTTP server (port 8081) + separate `StartCronjob()` entry
  - que_payment: long-running worker, switches behavior via `TYPE` and `PAYMENT_NAME` env

## Storage
- **MongoDB** — primary store (orders, statements, config, bank summary, ...) + event/error log collection `monitor_payment` (async batch 1s/1000 rows — `observability/mongodb/writer.go`)
- **Redis** — distributed lock (thorlock — ใช้เฉพาะ 3rd-payment), cache
- **RabbitMQ** — work queue, 1 queue / provider (`QUE_PAYMENT_<NAME>`), prefetch=1, manual ack
- ~~ClickHouse~~ — **ไม่มีจริง** (doc รุ่นก่อนเขียนผิด) — ไม่มี dependency ใน go.mod ทั้ง 2 repo; event log อยู่ใน MongoDB `monitor_payment`

## Observability
- **OpenTelemetry** via `github.com/Maximumsoft-Co-LTD/otelgo/eto`
  - HTTP middleware: W3C trace propagation + root span per request
  - AMQP: context propagation through message headers (`amqpcarrier.AmqpHeadersCarrier`)
- **Prometheus** — `metrics.NewPrometheus("gin")` (3rd-payment only)
- **Telegram alerting** — 2 chats: regular + on-call
  - severity 2-3 → regular chat, rate-limited 1 msg/s, dedup 5min
  - severity 4 → on-call chat (no rate-limit)
  - dedup key: `hash(severity + payment_code + error_type)`
- **Jaeger** — trace links embedded in Telegram alerts (env `JAEGER_URL` — `observability/telegram/notifier.go:69`)
- **Sentry** — que_payment เท่านั้น (`_cmd/main.go:33-38`, env `SENTRY_DSN`)

## External integrations
- payment provider APIs (each as its own folder) — 3rd-payment ~67 ตัว, que_payment 62 ตัว, ชื่อตรงกัน 59 ตัว
- Telegram Bot API (both business bot และ error bot)
- Optional: KBiz, SCB, BBL endpoints (ดูใน `bank-gateway/*` routes)

## Env vars สำคัญ (verify จากโค้ดจริง 2026-06-10)
```
PROJECT                                  # service name (used in traces, alerts)
PAYMENT_NAME                             # ใช้ใน que_payment, ระบุว่าเป็น provider ไหน
TYPE                                     # QUE_RABBIT | QUE_CRONJOB (que_payment)
RABBIT_MQ_HOST / RABBIT_MQ_PORT          # que_payment เท่านั้น — ฝั่ง 3rd-payment ดึง URL จาก
                                         # MongoDB config (serviceConfig.RabbitmqURL) และ
                                         # non-production hardcode amqp://guest:guest@127.0.0.1:5672
REDIS_URL / REDIS_DB / REDIS_PSW
DB_ENDPOINT / DB_DBNAME                  # MongoDB
OTEL_CONNECT                             # OTLP gRPC endpoint
JAEGER_URL                               # trace links ใน Telegram alerts
SENTRY_DSN                               # que_payment เท่านั้น
TELEGRAM_ERROR_BOT_TOKEN
TELEGRAM_ERROR_CHAT_ID
TELEGRAM_ERROR_ONCALL_CHAT_ID
BOT_TELEGRAM_TOKEN                       # 3rd-payment business bot
BOT_TELEGRAM_URL_WEBHOOK
BOT_TELEGRAM_SECRET
ALLOW_IP                                 # IP whitelist (3rd-payment)
AES_KEY / AES_IV                         # bank-gateway encryption (3rd-payment)
MODE                                     # production | (other)
```
> หมายเหตุ: `CLICKHOUSE_DSN` กับ `GRAFANA_TEMPO_URL` ใน doc รุ่นก่อนไม่มีจริงในโค้ด

## Distributed lock config
- `thorlock.InitLock(time.Second*180, time.Second*180, time.Second*10, 18)` — wait/lease 180s, retry 10s, DB 18 (`3rd-payment/app.go:85`)
- ⚠️ ใช้จริงเฉพาะ 3rd-payment — ฝั่ง que_payment มีแค่โค้ด comment ทิ้ง (`que_payment/services/bigpay/main.go:69`)
