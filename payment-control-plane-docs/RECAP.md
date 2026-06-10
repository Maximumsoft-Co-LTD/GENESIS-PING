# RECAP — Payment Control Plane

ไฟล์เดียวที่อ่านแล้วเข้าใจทุกอย่าง:
- (1) อะไรถูกสร้างไว้ใน docs repo นี้
- (2) DNA ของ `3rd-payment` + `que_payment` ทั้งคู่
- (3) จุดที่ 2 repo เชื่อมกัน + duplication ที่ต้องจัดการ
- (4) thesis ของการ rewrite

> ใช้ไฟล์นี้คู่กับ `REWRITE-PROMPT.md` — RECAP คือ context, PROMPT คือคำสั่ง

---

## ส่วนที่ 1 — สิ่งที่ Scaffold ไว้

### โครงสร้าง folder
```
payment-control-plane-docs/
├── README.md                       # entry
├── CLAUDE.md                       # คู่มือ Claude
├── RECAP.md                        # ไฟล์นี้
├── REWRITE-PROMPT.md               # prompt copy-paste ไปสั่ง AI
├── 00-overview/
│   ├── system-context.md           ✅ เขียนแล้ว
│   ├── glossary.md                 ✅ เขียนแล้ว
│   └── tech-stack.md               ✅ เขียนแล้ว
├── 01-source-repos/
│   ├── 3rd-payment/README.md       ✅ route map ครบ
│   └── que_payment/README.md       ✅ queue + cronjob behavior
├── 02-features/                    📭 ยังไม่ได้เติม — ใช้ template เติมทีหลัง
├── 03-workflows/                   📭
├── 04-integrations/                📭
├── 05-rewrite/                     📭
├── 06-adr/                         📭
└── templates/
    ├── service-doc.md              ✅
    ├── provider-doc.md             ✅
    ├── feature-doc.md              ✅
    ├── workflow.md                 ✅
    └── adr.md                      ✅
```

**ไฟล์ที่เติมเอง:** 11 ไฟล์ (README, CLAUDE, RECAP, REWRITE-PROMPT, overview×3, source-repos×2, templates×5)
**ที่เหลือ:** ใช้ template เติมแบบ incremental เมื่อ rewrite ลง feature ไหน

---

## ส่วนที่ 2 — DNA ของ `3rd-payment`

### บทบาท
**Front door** สำหรับระบบ payment — รับ HTTP จาก merchant และ webhook จาก provider

### Tech
- Go + Gin (port 8081)
- MongoDB (primary store), Redis (thorlock + cache), ClickHouse (event log), RabbitMQ (publish only)
- OTel ผ่าน `Maximumsoft-Co-LTD/otelgo`, Prometheus, Telegram error bot + business bot

### Surface area (จำง่ายเป็น 7 กลุ่ม)
1. **Order/Payin** — `payin`, `create-order`, `new-order-payin`, `callback-payin/:payment_code` (IP-whitelist), `callback-one-link/...`, `upload-slip-autopeer`
2. **Withdraw/Payout** — `create-withdraw`, `confirm-withdraw`, `reconfirm-withdraw`, `callback-payout/:payment_code` (+ variant `/autopeer`, `/umpay`, `/withdraw`), `cancel/payout`, `submit-withdraw`
3. **Verify/Recheck** — `virify/slip`, `check/order/:payment_code`, `check/ordercall/:payment_code/:service`, `confirm-transaction-payin*`
4. **Balance/Statement/Report** — `balance`, `check/balance`, `{deposit|topup|withdraw|payout}/statement[V2]`, `deposit/report[V2]`, `get-bank-summary/:service`
5. **Config/Admin** — `create-config-payment`, `update-config-payment`, `test-payment[v2]/:payment_code`, `get-bankconfig`
6. **Operations (office)** — `confirm-bank-deposit`, `cancel-bank-deposit`, `statement-deposit-not-confirm`, `corepay-user-*`, `register-bank-withdraw`
7. **Dashboard + Bot** — `dashboard/*`, `bot/webhook`, `bot/send-message`

### กลุ่ม route พิเศษ
- `/peer2pay/v3/*` — peer2pay-only namespace
- `/api/xpay/private/*` — xpay-only namespace
- `/bank-gateway/*` — bank-to-bank gateway (verify-transfer, confirm-transfer)
- `/call-pm/:paymentCode/*byPath` — **dynamic provider passthrough** (single most important pattern — ดูข้างล่าง)

### Pattern เด่นที่ฝังในโค้ด

| # | Pattern | ที่ไหน | ใช้ทำอะไร |
|---|---|---|---|
| 1 | **IP whitelist middleware** ก่อนทุก inbound callback | `middleware.IPWhitelist` | กัน provider ปลอม |
| 2 | **Distributed lock (thorlock)** wait/lease 180s, Redis DB 18 | `service/thorlock` | กัน concurrent payin/withdraw on same key |
| 3 | **One-link callback** — provider ยิง URL เดียว แล้ว branch by field | `callback-one-link/:payment_name/:payment_code` + `/deposit`, `/withdraw` | provider บางตัวไม่แยก endpoint |
| 4 | **Dynamic provider routing** | `/call-pm/:paymentCode/*byPath` → `controller.CallByPayment` | route ใดๆ ไปยัง provider โดยอิง `payment_code` |
| 5 | **Per-provider controller folder** | `controller/<name>/` ×60 | แต่ละ provider มี handler ของตัวเอง |
| 6 | **Side-by-side V1/V2** routes (`/statement` vs `/statementV2`) | ทั่ว routes/main.go | migration ที่ค้างคา — ทั้งคู่ live อยู่ |
| 7 | **Severity-based alerting** (1-4) → Telegram dedup 5min | `observability/telegram` + `classifier` | กัน alert flood |
| 8 | **Business Telegram bot** เป็น webhook | `bot/webhook` | merchant สั่งงานผ่าน Telegram (แยกจาก error bot) |

---

## ส่วนที่ 3 — DNA ของ `que_payment`

### บทบาท
**Worker** ที่ทำงาน async — กิน RabbitMQ + รัน cronjob

### Mode switch (env-driven)
1 image, behavior เปลี่ยนตาม env:
- `TYPE=QUE_RABBIT` → consumer
- `TYPE=QUE_CRONJOB` + `PAYMENT_NAME=BANK_SUMMARY_RECOVER` → `RecoverBankSummary`
- `TYPE=QUE_CRONJOB` + `PAYMENT_NAME=AUTOMATION_RETRY_CALLBACK_STATUS` → `AutomationRetryCallbackStatus`
- `TYPE=QUE_CRONJOB` + (else) → `StartUpdateRedis` + `ServRetryOrder` + `quepub.StartServV2` (3 goroutines)

### Message contract (RabbitMQ)
- Queue: `QUE_PAYMENT_<PAYMENT_NAME>` — **1 queue / 1 provider** (ไม่ใช่ 1 queue กลาง)
- prefetch=1, manual ack
- Body:
  ```json
  {"id": "...", "service": "...", "type": "...", "type_report": "...",
   "date": "...", "payment_code": "..."}
  ```
- Reply: `{"code": 0|1, "message": "..."}` พร้อม OTel trace context ใน AMQP headers

### Hybrid queue pattern
**สำคัญมาก** — ของพร้อมประมวลผลถูกหยิบได้ **2 ทาง**:
1. **RabbitMQ** `QUE_PAYMENT_<NAME>` → `StartAMQPV2WithContext` → `QueTypeSwitchV2`
2. **MongoDB poll** — `quepub.StartServV2` query collection `que_payment` ทุก 20 วินาที → worker pool 10 → `QueRunningV2`

→ MongoDB ทำหน้าที่เป็น "queue สำรอง / outbox" — ถ้า RabbitMQ ล่มหรือข้อความหาย ของยังคงรันได้

### Pattern เด่น

| # | Pattern | ที่ไหน | ใช้ทำอะไร |
|---|---|---|---|
| 1 | **Per-provider service folder** | `services/<name>/` ×60 — **ซ้ำกับ 3rd-payment** | ทำสิ่งเดียวกัน แต่อยู่คนละ repo |
| 2 | **Central type-switch dispatcher** | `QueTypeSwitchV2(ctx, repo, objReq, redisClient)` | branch by `objReq.Type` |
| 3 | **Hybrid queue (MQ + DB poll)** | rabbitmqpub + quepub | reliability ผ่าน 2 channel |
| 4 | **Worker pool with semaphore + timeout** | `ManageQueByServiceV2` (max 10, 30s) | bound concurrency per cronjob tick |
| 5 | **Error classifier** | `observability/classifier.Classify(err, source, ...)` | map error → severity + remediation hint |
| 6 | **Shutdown manager** | `shutdown.ShutdownManager` | coordinated graceful shutdown |
| 7 | **OTel context propagation through AMQP headers** | `amqpcarrier.AmqpHeadersCarrier` | trace ต่อเนื่อง HTTP → MQ |

---

## ส่วนที่ 4 — Boundary + Duplication

### จุดที่ 2 repo เจอกัน

```
                   3rd-payment                          que_payment
                   ───────────                          ───────────
HTTP request   ──▶ accept                                
                   │
                   ├──▶ MongoDB write   ◀─── poll ────  quepub.StartServV2 (20s)
                   │   (que_payment col)
                   │
                   └──▶ RabbitMQ publish ──────────▶   StartAMQPV2WithContext
                       (QUE_PAYMENT_<NAME>)            (consume + ack + reply)
                                                       │
                                                       ▼
                                                      QueTypeSwitchV2
                                                       │
                                                       ▼
                                                      services/<provider>/...  ◀── ⚠ ซ้ำกับ
                                                       │                         3rd-payment/controller/<provider>/
                                                       ▼
                                                      external bank/provider
                                                       │
                                                       ▼  result
                                                      MongoDB write back   ──▶  3rd-payment reads
                                                      (statement, bank_summary)

ทั้ง 2 repo → ClickHouse `payment_events` (async batch)
ทั้ง 2 repo → Telegram error bot (severity ≥ 2)
ทั้ง 2 repo ใช้ thorlock บน Redis DB 18 ตัวเดียวกัน
```

### Duplication ที่ต้องกำจัด

| What | 3rd-payment | que_payment | ปัญหา |
|---|---|---|---|
| Provider integration code | `controller/<name>/` | `services/<name>/` | **มี ~60 provider ซ้ำ 2 ที่ — แก้ที่นึงต้องไปแก้อีกที่** |
| MongoDB models | `model/*` (singular) | `models/*` (plural!) | naming inconsistent → import errors แฝง |
| Observability init | similar | similar | duplicate init code |
| Telegram notifier wiring | `helper.ErrorNotifier` | direct injection | binding pattern ไม่เหมือนกัน |
| MongoDB connection | `db.CreateConnection()` no ctx | `db.CreateConnection(ctx)` with ctx | API drift |
| Repository constructor | `repository.New(*conn)` | `repository.NewPayment(conn)` | naming/signature ต่าง |

### พฤติกรรมที่กระจัดกระจาย

| Behavior | 3rd-payment | que_payment |
|---|---|---|
| Receive payin/withdraw request | ✅ HTTP route | — |
| Receive inbound provider callback | ✅ HTTP route | — |
| Publish to queue | ✅ (ผ่าน MongoDB + Rabbit?) | — |
| Consume from queue | — | ✅ |
| Poll DB for stuck items | ✅ `CronjobStatement` (small) | ✅ `StartServV2` (heavy) |
| Outbound webhook to merchant | ✅ มี retry endpoint | ✅ `AutomationRetryCallbackStatus` |
| Update bank summary | ✅ (some) | ✅ `RecoverBankSummary` |

→ **เกือบทุก feature มีโค้ดอยู่ทั้ง 2 ฝั่ง** ในระดับใดระดับหนึ่ง

---

## ส่วนที่ 5 — Cross-cutting DNA (สำคัญที่สุดสำหรับ rewrite)

ถอด pattern ที่ใช้ซ้ำใน 60+ provider:

### ปฏิบัติการ ของแต่ละ provider (5 ดอก)
ทุก provider ต้อง implement ครบหรือบางส่วนของ:
1. **Payin / Create Order** — ขอ QR หรือ create payment intent
2. **Inbound Callback (webhook)** — provider แจ้งสถานะ
3. **Payout / Create Withdraw** — สั่งโอน
4. **Outbound Callback (เราแจ้ง merchant)** — ผ่าน OfficeAPI
5. **Query / Reconcile** — เช็คสถานะ order ย้อนหลัง
6. **Balance / Bank Summary** — เช็คยอดในบัญชี/wallet

### Variation points (จุดที่ provider ต่างกัน)
- **Auth method** — HMAC vs API key vs JWT vs mTLS
- **Endpoint shape** — split endpoints vs one-link callback
- **Signature scheme** — fields signed, order, encoding
- **Idempotency** — provider-supplied vs merchant-generated key
- **Time format** — UTC ISO vs +07 vs unix
- **Response shape** — flat vs nested vs string-encoded JSON
- **Error semantics** — HTTP 200 with `code:1` vs HTTP 4xx
- **Retry attitude** — provider retry callback ซ้ำกี่ครั้ง, ดีเลย์เท่าไหร่

### Cross-cutting concerns ที่ใช้ทุกที่
- **Idempotency key**: `payment_code + order_no`
- **Distributed lock**: thorlock key per `payment_code + order_no` ก่อน mutate state
- **Error classification + severity 1-4** → Telegram
- **OTel span**: `payment_code`, `queue_type` เป็น standard attribute
- **ClickHouse `payment_events`**: ทุก error เขียนพร้อม trace_id, span_id
- **MongoDB transaction boundary**: เกือบไม่ใช้ — เปลี่ยน state แล้วยิง downstream

---

## ส่วนที่ 6 — Thesis ของการ Rewrite

ถ้าจะถอด DNA ออกมา rewrite ให้ดี อย่างน้อยต้องตัดสินใจ 7 เรื่องนี้:

1. **One repo or two?**
   - รวมเป็น monorepo (ลด duplication แต่ deployment ยากขึ้น)
   - หรือแยกเป็น `gateway` (HTTP) + `worker` (queue/cron) ที่ share `provider-adapters` package

2. **Provider abstraction** — ต้องมี interface เดียวที่ทุก provider implement
   ```go
   type Provider interface {
       CreatePayin(ctx, PayinReq) (PayinRes, error)
       CreatePayout(ctx, PayoutReq) (PayoutRes, error)
       VerifyCallback(ctx, raw []byte, headers) (Event, error)
       QueryOrder(ctx, ref) (OrderStatus, error)
       Balance(ctx) (Balance, error)
   }
   ```
   เพื่อทำลาย duplication ระหว่าง `controller/<name>` กับ `services/<name>`

3. **Queue strategy** — เก็บ "MQ + DB poll" pattern ไว้ไหม?
   - Keep → reliability ดี, complexity สูง
   - Drop → ใช้ MQ อย่างเดียว + dead-letter queue
   - **แนะนำ**: เปลี่ยนเป็น Outbox pattern (write DB tx + publish ใน tx เดียว)

4. **Idempotency** — ทำให้เป็น first-class
   - Idempotency-Key header → store ใน Redis with TTL
   - `payment_code + order_no` เป็น natural key
   - **ทุก** callback ต้องผ่าน idempotency check, ไม่ใช่บาง provider

5. **Webhook delivery** — ของ outbound (เราส่งให้ merchant) ต้องมี
   - retry with exponential backoff
   - signature ของเราเอง (อย่าฝากให้ merchant trust IP)
   - dead-letter ที่ดูได้

6. **Config plane** — ไฟล์โค้ดไม่ควรรู้จัก provider name ตรงๆ
   - `config_payment` ใน DB เก็บ provider type + credentials + flags
   - Adapter factory โหลด config + return implementation
   - การเพิ่ม provider ใหม่ = เพิ่ม row ใน config + register adapter, ไม่ใช่แก้ route

7. **Observability** — เก็บ pattern เดิม (OTel + ClickHouse + Telegram severity) แต่แยก library ออกเป็น shared module

---

## ส่วนที่ 7 — Non-negotiables (ของห้ามทิ้ง)

จากการอ่าน code, สิ่งเหล่านี้ทำงานได้ดีอยู่แล้ว อย่าตัดทิ้งโดยไม่มีเหตุผล:

- ✅ **OTel propagation ผ่าน AMQP headers** (amqpcarrier) — เก็บ
- ✅ **Severity-based Telegram alerting + dedup 5min** — เก็บ
- ✅ **Shutdown manager pattern** ของ que_payment — ขยายไปใช้ทั้งระบบ
- ✅ **Per-provider isolation** (1 queue / 1 provider) — ป้องกัน noisy neighbor
- ✅ **Distributed lock ก่อน mutate** — เก็บ แต่ wrap ให้สั้นลง
- ✅ **IP whitelist middleware** สำหรับ inbound callback — เก็บ
- ✅ **ClickHouse async batch writer** (1s/1000 rows) — เก็บ pattern

---

## ส่วนที่ 8 — Kill list candidates (ของน่าทิ้ง)

ต้อง confirm ทุกอันก่อนทิ้ง — แต่นี่คือ candidate ที่เห็นชัดจาก code:

- ❌ **Duplicate provider code ใน 2 repo** — รวมเป็น adapter package เดียว
- ❌ **V1/V2 route side-by-side** ที่ยัง live ทั้งคู่ — เลือก V2, ลบ V1 (ต้อง verify caller)
- ❌ **Provider-specific URL variants** (`/callback-payout/autopeer/...`, `/callback-payout/umpay/...`, `/callback-payout/.../withdraw` for hengpay) — รวมเป็น 1 URL + signature/header-based dispatch
- ❌ **MongoDB-as-queue polling** ถ้าทำ Outbox pattern แล้ว
- ❌ **Inconsistent naming** (`model` vs `models`, `repository.New` vs `repository.NewPayment`)
- ❌ **`/api/v2/get-statement-withdraw-by-orderNo` GET + POST batch variant** — เก็บ POST อย่างเดียว
- ❌ **Inline test/debug endpoints** (`/update-report-test`, `/filter-report-test2`) ใน production routes
- ❌ **Commented-out code** ใน routes (RouteBankTransferGateway มี comment block ใหญ่)

---

## ส่วนที่ 9 — Open questions ที่ต้องถาม user ก่อน rewrite

(prompt ใน `REWRITE-PROMPT.md` จะถามคำถามเหล่านี้แทนคุณ)

1. **Indonesia variant** (`create-indo-config-payment`) เป็น product แยกหรือแค่ flag?
2. **Peer2Pay namespace** (`/peer2pay/v3`) เป็น merchant-specific หรือ provider-specific?
3. **Business Telegram bot** กับ error bot — บทบาทต่างกันแน่ไหม, ใครใช้ business bot?
4. **Corepay** — เป็น provider ปกติหรือมี user/wallet management แยก?
5. **bank-gateway namespace** — ใช้กับธนาคารตรง (KBiz/SCB) หรือผ่าน provider?
6. **Cronjob `CronjobStatement` ใน 3rd-payment** vs `que_payment` cronjobs — ทับซ้อนกันแค่ไหน?
7. ปริมาณ traffic / throughput ที่ต้องรองรับ?
8. ทีมที่ใช้งานปัจจุบันใหญ่แค่ไหน — มี on-call ไหม?
