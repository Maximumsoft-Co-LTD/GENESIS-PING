# 3rd-payment

HTTP API + internal cronjob (Gin, Go)

## Entry points
- `app.go:StartServ()` — start HTTP server on port 8081
- `app.go:StartCronjob()` — start `controller.CronjobStatement(repo)`

## Init order ที่ StartServ
1. ตั้งค่า Telegram **business bot** webhook (`/api/v2/bot/webhook`) ถ้ามี `BOT_TELEGRAM_TOKEN`
2. Connect MongoDB (`db.CreateConnection`)
3. Init **thorlock** (Redis distributed lock, wait/lease 180s, DB 18)
4. Init **OpenTelemetry** (`observability.Init`)
5. Init **Telegram error notifier** (separate from business bot)
6. Init **MongoDB writer** (writes to ClickHouse `payment_events`)
7. Wire `helper.ErrorNotifier` และ `helper.EventWriter`
8. Init **Prometheus** middleware (`metrics.NewPrometheus("gin")`)
9. Mount **OTel middleware**
10. Mount routes:
    - `routes.RouteManage` — main `/api/v2/*`
    - `routes.RouteBankTransferGateway` — `/bank-gateway/*`
    - `routes.RouteByPeer2Pay` — `/peer2pay/v3/*`
    - `routes.RouteXpayPrivate` — `/api/xpay/private/*`
    - `routes.NewRouteCallByPayment("call-pm")` — dynamic `/call-pm/:paymentCode/*byPath`

## Route map (highlights — เต็มอยู่ที่ `routes/main.go`)

### Order / Payin
- `POST /api/v2/payin` — entry สำหรับสร้าง deposit
- `POST /api/v2/new-order-payin`, `POST /api/v2/create-order`
- `GET /api/v2/get-order/:order_no`
- `POST /api/v2/callback-payin/:payment_code` (IP-whitelist) — inbound callback
- `POST /api/v2/callback-one-link/:payment_name/:payment_code` (+ /deposit, /withdraw)
- `POST /api/v2/upload-slip-autopeer`

### Withdraw / Payout
- `POST /api/v2/create-withdraw`
- `POST /api/v2/confirm-withdraw`, `/reconfirm-withdraw`
- `POST /api/v2/callback-payout/:payment_code` (IP-whitelist)
  - special variants: `/autopeer`, `/umpay`, `/:payment_code/withdraw` (hengpay)
- `POST /api/v2/check-uuid/umpay/:payment_code/:uuid`
- `POST /api/v2/refund-withdraw-payment`
- `POST /api/v2/cancel/payout`
- `POST /api/v2/submit-withdraw`

### Verify / Recheck
- `POST /api/v2/virify/slip` (first2pay)
- `POST /api/v2/callback/verify_slip/:payment_code/:order_no`
- `POST /api/v2/check/order/:payment_code`
- `POST /api/v2/check/ordercall/:payment_code/:service`
- `POST /api/v2/check/neworder/:payment_code/:order`
- `POST /api/v2/confirm-transaction-payin` / `-slip`

### Balance / Statement
- `POST /api/v2/balance`
- `GET /api/v2/check/balance`
- `POST /api/v2/{deposit,topup,withdraw,payout}/statement`
- `POST /api/v2/{deposit,topup,withdraw,payout}/statementV2`
- `POST /api/v2/deposit/report` / `reportV2`
- `GET /api/v2/get-bank-summary/:service`
- `POST /api/v2/update-active-status-bank-summary`

### Config / Admin
- `POST /api/v2/create-config-payment` / `update-config-payment`
- `POST /api/v2/create-indo-config-payment` / `update-indo-config-payment` (Indonesia variant)
- `GET /api/v2/get-config-payment/:payment_code`
- `GET /api/v2/test-payment/:payment_code`
- `POST /api/v2/test-payment-v2/:payment_name`

### Corepay (user account)
- `POST /api/v2/corepay-user-register` / `-cancel` / `-register-all`

### Office (manual operations)
- `POST /api/v2/confirm-bank-deposit` / `cancel-bank-deposit`
- `POST /api/v2/statement-deposit-not-confirm`

### Dashboard
- `GET /api/v2/dashboard/balancePayment` / `:payment_name`
- `POST /api/v2/dashboard/dashboard`, `/transaction`, `/profit`

### Bot
- `POST /api/v2/bot/log-telegram-list`, `send-message`, `update-log-telegram`, `webhook`

### Dynamic provider passthrough
- `ANY /call-pm/:paymentCode/*byPath` → `controller.CallByPayment` (route ไปยัง provider ตาม payment_code)

## Folder layout
```
controller/        per-provider controllers (60+ folders) + cross-cutting handlers
                   (deposit.go, withdraw.go, callback.go, Balance.go, VerifySlip.go, ...)
controller/dashboard/
controller/banktransfer/
service/           cross-cutting services (callback, checkBank, genQrImage, verifySlip,
                   thorlock, hashlayout, paymentIndo, gstark, opentelemetry, otelPatternV2)
routes/            route registration (main.go, bank-transfer-gateway.go, call-by-payment.go)
middleware/        IPWhitelist, OTel, CORS, etc.
model/             MongoDB models
repository/        data access layer
observability/     mongodb writer, telegram notifier, OTel init
helper/            shared helpers
metrics/           Prometheus
db/                Mongo connection
_cmd/              cmd entry
```

## ของที่น่าจับตามอง (DNA candidates)
- **`controller.CallByPayment` + `/call-pm/:paymentCode/*byPath`** — โครงสร้างที่ "route ไปยัง provider แบบ dynamic" — เป็นต้นแบบของ provider abstraction ที่ rewrite ใหม่ควรขยายให้เป็นทุก op ไม่ใช่แค่ passthrough
- **`service/thorlock`** — distributed lock pattern ที่ใช้ทั่วทั้ง 2 repo
- **`middleware.IPWhitelist`** — ใช้ก่อนทุก inbound callback
- **`/api/v2/callback-one-link/...`** — pattern "one URL, branch by field" → indicator ว่า provider บางตัวไม่แยก endpoint deposit/withdraw
