# que_payment

RabbitMQ consumer + cronjob worker (Go)

## Entry point
- `app.go:StartServ(sm *shutdown.ShutdownManager)` — switches by `TYPE` env

## 2 modes via env `TYPE`

### `TYPE=QUE_RABBIT`
- Connect RabbitMQ via `rabbitmqpub.ConnectRabbitV2`
- Start consumer goroutine → `rabbitmqpub.StartAMQPV2WithContext`
- Queue: `QUE_PAYMENT_<PAYMENT_NAME>` (1 queue / 1 provider)
- Prefetch=1, manual ack, no exchange (direct to queue)
- Message contract:
  ```go
  type ReqQue struct {
      ID          string `json:"id"`
      Service     string `json:"service"`
      Type        string `json:"type"`
      TypeReport  string `json:"type_report"`
      Date        string `json:"date"`
      PaymentCode string `json:"payment_code"`
  }
  type ResQue struct {
      Code    int    `json:"code"`     // 0=success, 1=error
      Message string `json:"message"`
  }
  ```
- Reply published with OTel trace context in AMQP headers
- Error classification via `observability/classifier.Classify(...)` → writes to ClickHouse + sends Telegram alert if severity ≥ 2
- Auto-reconnect via `Reconnector()` goroutine

### `TYPE=QUE_CRONJOB`
Behavior switches by `PAYMENT_NAME`:

| PAYMENT_NAME | Behavior |
|---|---|
| `BANK_SUMMARY_RECOVER` | `cronjob.RecoverBankSummary` |
| `AUTOMATION_RETRY_CALLBACK_STATUS` | `cronjob.AutomationRetryCallbackStatus` |
| (anything else / default) | runs 3 goroutines in parallel: `cronjob.StartUpdateRedis`, `cronjob.ServRetryOrder`, `quepub.StartServV2` |

### `quepub.StartServV2`
- Polls MongoDB collection `que_payment` every **20 seconds**
- `repo.GetListQueAllV2(PAYMENT_NAME)` returns pending items
- `ManageQueByServiceV2` runs them with worker pool (max **10 workers**, **30s timeout**)
- Each item → `QueRunningV2(repo, obj)`
- This is **fallback / safety net** for RabbitMQ — items can be processed via either channel

## Init order
1. OTel init (`observability.Init`)
2. Telegram notifier
3. MongoDB connection (timeout 30s)
4. MongoDB writer (ClickHouse `payment_events`)
5. Redis connection (timeout 30s)
6. Branch by `TYPE`

## Folder layout
```
services/          per-provider services (60+ folders) — ซ้ำกับ 3rd-payment/controller/
                   (anypay, askmepay, maanpay, peer2pay, ...)
controllers/       cross-cutting controllers
cronjob/           RecoverBankSummary, AutomationRetryCallbackStatus, StartUpdateRedis,
                   ServRetryOrder, _retry.go
quepub/            cornjob.go — polls MongoDB que_payment collection
rabbitmqpub/       amqp.go (consumer), conn.go (connection), amqpcarrier/ (OTel headers)
service/           sudahpay (orphan?)
models/            MongoDB models (note: `models/` plural vs 3rd-payment's `model/`)
repository/
redis/
db/                Mongo connection
observability/     mongodb writer, telegram notifier, classifier, OTel
shutdown/          ShutdownManager (graceful shutdown coordinator)
_cmd/              cmd entry
```

## ของที่น่าจับตามอง (DNA candidates)
- **Hybrid queue (RabbitMQ + Mongo poll)** — quepub.StartServV2 polls MongoDB ทุก 20s เป็น fallback → bug/feature?
- **`QueTypeSwitchV2`** — central dispatcher ที่ branch by `objReq.Type` → เป็น "router" ของ worker
- **Per-provider service ซ้ำกับ 3rd-payment** — payment_code อันเดียวกัน, ทำสิ่งเดียวกัน, แต่คนละ repo → คือ pain point หลัก
- **Cronjob branching ด้วย `PAYMENT_NAME` env** — ทำให้ deploy 1 image แต่หลายบทบาท → ขึ้นอยู่กับว่าตอน deploy ตั้ง env อะไร
- **`shutdown.ShutdownManager`** — pattern ที่ดี — rewrite ใหม่ควรคงไว้
