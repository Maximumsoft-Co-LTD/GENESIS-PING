---
last_reviewed: 2026-06-10
---

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
6. Init **MongoDB event writer** — เขียน event/error ลง MongoDB collection `monitor_payment` (`observability/mongodb/writer.go:17`, async batch 1s/1000 rows)
7. Wire `helper.ErrorNotifier` และ `helper.EventWriter`
8. Init **Prometheus** middleware (`metrics.NewPrometheus("gin")`)
9. Mount **OTel middleware**
10. Mount routes:
    - `routes.RouteManage` — main `/api/v2/*`
    - `routes.RouteBankTransferGateway` — `/bank-gateway/*`
    - `routes.RouteByPeer2Pay` — `/peer2pay/v3/*`
    - `routes.RouteXpayPrivate` — `/api/xpay/private/*`
    - `routes.NewRouteCallByPayment("call-pm")` — dynamic `/call-pm/:paymentCode/*byPath`

## Route map (sync กับ `routes/main.go` เมื่อ 2026-06-10)

### Order / Payin
- `POST /api/v2/payin` — entry สำหรับสร้าง deposit
- `POST /api/v2/new-order-payin`, `POST /api/v2/create-order`
- `GET /api/v2/get-order/:order_no`
- `POST /api/v2/get-order-transaction`
- `POST /api/v2/callback-payin/:payment_code` (IP-whitelist) — inbound callback
- `POST /api/v2/callback-payin/:payment_code/callback/qrStatus` (IP-whitelist, peer2pay only)
- `POST /api/v2/callback-one-link/:payment_name/:payment_code` (+ /deposit, /withdraw) (IP-whitelist)
- `POST /api/v2/callback-transaction/:payment_code`
- `POST /api/v2/upload-slip-autopeer`

### Withdraw / Payout
- `POST /api/v2/create-withdraw`
- `POST /api/v2/confirm-withdraw`, `/reconfirm-withdraw`
- `POST /api/v2/withdraw/payment`
- `POST /api/v2/callback-payout/:payment_code` (IP-whitelist)
  - special variants: `/autopeer`, `/umpay`, `/:payment_code/withdraw` (hengpay)
- `POST /api/v2/check-uuid/umpay/:payment_code/:uuid`
- `POST /api/v2/refund-withdraw-payment`
- `POST /api/v2/cancel/payout`
- `POST /api/v2/submit-withdraw`
- `POST /api/v2/get-payment-auto-withdraw`

### Verify / Recheck
- `POST /api/v2/virify/slip` (first2pay — typo "virify" อยู่ในโค้ดจริง `routes/main.go:135`)
- `POST /api/v2/callback/verify_slip/:payment_code/:order_no`
- `POST /api/v2/check/order/:payment_code`
- `POST /api/v2/check/ordercall/:payment_code/:service`
- `POST /api/v2/check/neworder/:payment_code/:order` (hengpay ไม่ใช้)
- `POST /api/v2/confirm-transaction-payin` / `-slip` (manual confirm จาก office)
- `POST /api/v2/change-status-deposit-payment`
- `POST /api/v2/request-resend-callback/:payment_code/:order` (jaijaipay)

### Balance / Statement / Report
- `POST /api/v2/balance`, `GET /api/v2/check/balance`
- `POST /api/v2/adjust-balance`
- `GET /api/v2/get-deposit`
- `POST /api/v2/{deposit,topup,withdraw,payout}/statement` (V1)
- `POST /api/v2/{deposit,topup,withdraw,payout}/statementV2` (V2 — live คู่กับ V1)
- `POST /api/v2/username/statement`
- `GET /api/v2/get-statement-withdraw/:payment_code`
- `GET /api/v2/get-statement-withdraw-by-orderNo/:orderNo` + `POST` batch variant (kill-list candidate)
- `POST /api/v2/re-create-statement/:service/:paymentCode/:orderNo`
- `GET /api/v2/get-payment-transection-detail/:orderNo`
- `POST /api/v2/deposit/report` / `reportV2`
- `POST /api/v2/report/payment`, `/update/report-payment`, `/update-old/report-payment`
- `GET /api/v2/get-bank-summary/:service`
- `POST /api/v2/update-active-status-bank-summary`
- ⚠️ test/debug ใน prod: `GET /api/v2/update-report-test`, `GET /api/v2/filter-report-test2` (kill list)

### Config / Admin
- `POST /api/v2/create-config-payment` / `update-config-payment`
- `POST /api/v2/create-indo-config-payment` / `update-indo-config-payment` (Indonesia variant)
- `GET /api/v2/get-config-payment/:payment_code`
- `POST /api/v2/get-payment-list-by-name`, `/update-payment-list-by-payment-code`
- `POST /api/v2/check-payment-config/:payment_code`
- `GET /api/v2/test-payment/:payment_code`, `POST /api/v2/test-payment-v2/:payment_name`
- `POST /api/v2/get-bankconfig`, `/update-bankconfig`
- `POST /api/v2/get-bank-support` (+ `/clear-redis`), `/token-bigpay/clear-redis`

### Corepay (user account)
- `POST /api/v2/create-member-data` (เรียก `controller.CronjobStatement` ตรงๆ!)
- `POST /api/v2/corepay-user-register` / `-cancel` / `-register-all`

### Office (manual operations)
- `POST /api/v2/confirm-bank-deposit` / `cancel-bank-deposit` (status 99)
- `POST /api/v2/update-active-bank-deposit`
- `POST /api/v2/statement-deposit-not-confirm`
- `POST /api/v2/retry-payment-sendffice` (retry outbound webhook — typo "sendffice" อยู่ในโค้ดจริง)

### Hengpay-specific
- `POST /api/v2/update-callback`, `/register-bank-withdraw`, `/update-bank-withdraw`

### Peer2Pay-specific (ใน /api/v2 — นอกเหนือจาก namespace /peer2pay/v3)
- `POST /api/v2/create-p2p-qrpayment`, `/get-bankconfig-p2p`, `/status-p2p-qrpayment`
- `POST /api/v2/get-tranfer-history-peer2pay`, `/check_balance_p2p`, `/existing-uid`
- `GET /api/v2/p2cpay/channels/:payment_code`

### Dashboard
- `GET /api/v2/dashboard/balancePayment` + `POST` variant (Beepay)
- `GET /api/v2/dashboard/balancePaymentByCs/:payment_name` (CS)
- `POST /api/v2/dashboard/dashboard`, `/transaction`, `/profit`

### Bot
- `POST /api/v2/bot/log-telegram-list`, `send-message`, `update-log-telegram`, `webhook`

### Namespace แยก
- `/peer2pay/v3/*` — payin, callback-payin (+qrStatus), callback-payout, create/confirm-withdraw, p2p endpoints (`routes/main.go:164-179`)
- `/api/xpay/private/*` — callback-payin เท่านั้น (`routes/main.go:181-187`)
- `/bank-gateway/*` — bank gateway CRUD + verify-transfer, confirm-transfer (`routes/bank-transfer-gateway.go` — mount ที่ `app.go:144`; เวอร์ชัน comment ทิ้งใน `routes/main.go:189-210` คือซาก)

### Dynamic provider passthrough
- `ANY /call-pm/:paymentCode/*byPath` → `controller.CallByPayment` (route ไปยัง provider ตาม payment_code)

## Folder layout
```
controller/        per-provider controllers (~67 folders) + cross-cutting handlers
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
- **`service/thorlock`** — distributed lock pattern — ⚠️ ใช้จริงเฉพาะ repo นี้ (ฝั่ง que_payment ถูก comment ทิ้ง — worker ไม่มี lock)
- **`middleware.IPWhitelist`** — ใช้ก่อนทุก inbound callback
- **`/api/v2/callback-one-link/...`** — pattern "one URL, branch by field" → indicator ว่า provider บางตัวไม่แยก endpoint deposit/withdraw
