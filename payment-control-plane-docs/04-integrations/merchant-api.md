---
name: merchant-api
type: integration
status: active
last_reviewed: 2026-06-10
---

# Merchant-Facing API Contract

> **Contract ที่ห้ามพัง** — merchant integrate กับ schema เหล่านี้อยู่ การ rewrite ต้อง backward-compatible หรือมี versioning plan

## Response Envelope (ทุก endpoint)

`3rd-payment/helper/resp.go:26-39`

```json
{ "code": 0|1|4xx, "msg": "string (มักเป็นไทย)", "data": <payload>, "error": "string" }
```
- `code` 0 หรือ 1 = สำเร็จ (convention ไม่คงที่ — บาง endpoint ใช้ 0, บางใช้ 1, บางใช้ 200) ⚠️ rewrite ควร normalize แต่ต้องคง wire format
- HTTP status มัก 200 เสมอแม้ error (ดู code ใน body)

## Authentication

- **ไม่มี API key/JWT บน endpoint หลัก** — ใช้ **service whitelist**: เช็คว่า `service` (ชื่อ merchant) ถูก whitelist กับ `payment_code` นั้นใน config (`middleware/service_whitelist.go`, เรียกจาก `deposit.go:135`, `withdraw.go:95`)
- `access_key`/`secret_key` ใน body — ใช้เฉพาะบาง provider (เช่น PEER2PAY)
- IP whitelist ใช้กับ **callback ขาเข้าจาก provider เท่านั้น** ไม่ใช่กับ merchant

## Endpoints หลัก

### POST /api/v2/payin — `controller/deposit.go:103`

Request `ReqDeposit` (`controller/model.go:11-57`) — required: `username`, `service`, `office_api`, `payment_code`, `customer_api`:
```
username, phone_number, operator_name, service*, amount, payer_account_no,
payer_account_code, payer_account_first_name, payer_account_last_name,
office_api*, payment_code*, network, currency, customer_api*, scope,
deposit_payment_type (BANK|QR), access_key, secret_key, payment_name, datetime
```

Response `RespDeposit`:
```
username, orderNo, amount_actual, url_qrcode, new_order (bool — false = reuse order เดิม),
deposit_payment_type, countdown_time (วินาที), message,
bank_number, bank_code, bank_name, payment_name, acc_bank_number, acc_bank_code, acc_bank_name
```

### POST /api/v2/create-withdraw — `controller/withdraw.go:38`

Request `ReqCreateWithdraw` (`controller/model.go:80-97`) — required: `datetime`, `amount`, `bank_code`, `bank_name`, `bank_number`, `channel` (WITHDRAW|PAYOUT), `service`, `office_api`, `payment_code`
Optional: `phone_number`, `network`, `currency`, `access_key`, `secret_key`, `refund` (bool), `order_deposit`

Response: `data` = orderNo (string เปล่าๆ) — merchant ต้องเก็บไว้ใช้ confirm

### POST /api/v2/confirm-withdraw — `controller/withdraw.go:308`

Request: `{orderNo, payment_code, access_key?, secret_key?}` → Response: `data: null`, msg `"SUCCES"` (typo อยู่ใน contract จริง)
ถ้า order อยู่สถานะ 15/16 (รอ provider ตอบ) จะได้ msg `"WAITWITHDRAW"` — **ไม่ใช่ error**

### POST /api/v2/balance — `controller/Balance.go:74`

Request `{payment_code*, type_balance}` → Response `data` = **รูปร่างต่างกันตาม provider** (float, object) ⚠️ rewrite ควร normalize

### GET /api/v2/check/balance — `controller/main.go:718`
Query: `payment_code`, `service` → `data` = string `"%.2f"`

### POST /api/v2/{deposit,topup,withdraw,payout}/statement[V2]

- V1 Request `ReqPayment` (`model/bank_summary.go:36-43`): `{service, payment_code (string), date, from, limit, page}`
- V2 Request `ReqPaymentV2` (`model/bank_summary.go:106-113`): เหมือน V1 แต่ `payment_code` เป็น **array**
- `from`: WITHDRAWPAYMENT = เดือน, PAYOUT = วัน (semantics ฝังใน field เดียว ⚠️)

### POST /api/v2/check/order/:payment_code — `controller/transaction.go:494`
Request `{orderid, type_order, custName}` → Response `ResCheckOrder`: `{service, orderid, amount, payment_code, status (int32), create_at}`

### POST /api/v2/create-order — `controller/order.go:242` / POST /api/v2/new-order-payin — `order.go:39`
Request (ทุก field required ยกเว้น scope): `{username, service, amount, account_no, account_code, account_name, payment_code, ref_no (create-order เท่านั้น), office_api, scope?}`
Response: `{order_no, payment_code, amount_create, status (create-order เท่านั้น)}`

## ข้อสังเกตสำหรับ rewrite

- 🔴 **ไม่มี merchant authentication จริง** — whitelist by service name เท่านั้น → ควรเพิ่ม API key/HMAC (เป็น design goal อยู่แล้ว)
- 🟡 `code` convention ไม่คงที่ (0/1/200 = สำเร็จ แล้วแต่ endpoint) — normalize ภายใน แต่คง wire compat ต่อ endpoint
- 🟡 `balance` response ไม่ normalize ข้าม provider
- Idempotency ฝั่ง payin: duplicate check by `(payment_code, service, username, amount)` คืน order เดิม (`new_order: false`) — ฝั่ง withdraw: hash `(service+banknumber+date+amount)` ห้ามซ้ำ **ภายในวัน** (`withdraw.go:282`)
