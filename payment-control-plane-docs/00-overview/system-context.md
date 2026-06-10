# System Context

## ใครคุยกับใคร

```
                       ┌─────────────────────┐
   Merchant (web/app)──▶│   3rd-payment API   │◀── Bank/Provider callback
                       │  (Gin, port 8081)   │
                       └──────┬──────────────┘
                              │
                       ┌──────┴───────────┬────────────────┐
                       ▼                  ▼                ▼
                ┌────────────┐    ┌─────────────┐   ┌──────────────┐
                │ MongoDB    │    │   Redis     │   │  RabbitMQ    │
                │ (primary)  │    │ (lock/cache)│   │  (work queue)│
                └────────────┘    └─────────────┘   └──────┬───────┘
                       ▲                                   │
                       │                                   ▼
                       │                          ┌────────────────────┐
                       │                          │   que_payment      │
                       │                          │ (RabbitMQ consumer │
                       └──────────────────────────┤  + 3 cronjob mode) │
                                                  └────────┬───────────┘
                                                           │
                                                           ▼
                                              ┌──────────────────────┐
                                              │  Provider/Bank API   │
                                              │  (60+ adapters)      │
                                              └──────────────────────┘
                       ┌─────────────────────┐
                       │   ClickHouse        │◀── เขียน event/error
                       │  payment_events     │    จากทั้ง 2 repo
                       └─────────────────────┘
                       ┌─────────────────────┐
                       │   Telegram Bot      │◀── alert severity ≥ 2
                       │  (error + on-call)  │    จากทั้ง 2 repo
                       └─────────────────────┘
```

## ขอบเขตของแต่ละ repo

### 3rd-payment
- **Front door** — รับ request จาก merchant และ callback จาก provider
- มี **per-provider controller** ของตัวเอง (~60 ตัว) ใน `controller/<name>/`
- มี HTTP route หลายกลุ่ม: `/api/v2/*`, `/peer2pay/v3/*`, `/api/xpay/private/*`, `/bank-gateway/*`, `/call-pm/*`
- มี dynamic provider route: `/call-pm/:paymentCode/*byPath` (route ตาม `payment_code` ตรงๆ)
- มี internal cronjob: `StartCronjob` → `CronjobStatement`
- มี Telegram **business bot** (webhook สำหรับลูกค้าใช้สั่งงาน)

### que_payment
- **Worker** — กิน RabbitMQ + รัน cronjob 3 แบบ
- มี **per-provider service** ของตัวเอง (~60 ตัว) ใน `services/<name>/` — **ซ้ำกับ 3rd-payment**
- 2 mode ผ่าน env `TYPE`:
  - `QUE_RABBIT` — consumer ของ `QUE_PAYMENT_<PAYMENT_NAME>`
  - `QUE_CRONJOB` — มี 3 sub-mode ตาม `PAYMENT_NAME`:
    - `BANK_SUMMARY_RECOVER` — `cronjob.RecoverBankSummary`
    - `AUTOMATION_RETRY_CALLBACK_STATUS` — `cronjob.AutomationRetryCallbackStatus`
    - (default) — `StartUpdateRedis` + `ServRetryOrder` + `quepub.StartServV2` (poll MongoDB `que_payment` collection ทุก 20s, worker pool 10)

## Boundary ระหว่าง 2 repo

จุดที่ 2 repo เจอกัน:

| Touch point | Direction | Contract |
|---|---|---|
| RabbitMQ queue `QUE_PAYMENT_<NAME>` | 3rd → que | `ReqQue{ID, Service, Type, TypeReport, Date, PaymentCode}` → `ResQue{Code, Message}` |
| MongoDB `que_payment` collection | 3rd writes → que reads | poll mode fallback |
| MongoDB shared collections | both R/W | `statement`, `bank_summary`, `service_payment`, etc. |
| ClickHouse `payment_events` | both write | async batch (1s / 1000 rows) |
| Telegram alert | both publish | severity 1-4 |

## Stakeholder

- **Dev team** — เจ้าของ rewrite
- **Office/Operations** — ใช้ dashboard, manual confirm/cancel deposit
- **Merchant developer** — เรียก API
- **Provider** — ยิง callback เข้ามา
