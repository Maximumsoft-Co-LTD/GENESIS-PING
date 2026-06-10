---
name: office-webhook
type: integration
status: active
last_reviewed: 2026-06-10
---

# Outbound Webhook → Merchant Office

> Contract ที่เรายิงแจ้ง merchant (office) เมื่อรายการจบ — อีก contract ที่ห้ามพังตอน rewrite

## URL ปลายทาง (ต่อจาก `office_api` ที่ merchant ลงทะเบียน)

| ประเภท | URL | payload |
|---|---|---|
| Deposit | `{OfficeAPI}/api/Payment/PaymentCreateStatement/{service}` | bson.M (ไม่มี struct!) |
| Withdraw | `{OfficeAPI}/api/WithdrawCallback` | `DataCallbackWithdraw` |
| Withdraw (amount variant) | `{OfficeAPI}/api/Payment/CallBackWithdrawAmount/{service}` | เหมือน Withdraw |
| Payout | `{OfficeAPI}/api/WithdrawCallback` (URL เดียวกับ withdraw) | payload คนละ shape |

## Payload

### Deposit — `3rd-payment/controller/transaction.go:2128-2149`, `que_payment/cronjob/automation_retry_callback_status.go:198-220`
สร้างเป็น `bson.M` สดๆ (**ไม่มี Go struct กำกับ** ⚠️ — สอง repo ประกอบเองคนละที่ เสี่ยง drift):
```
_id (ObjectID ใหม่ทุกครั้ง!), datetime, Date ("2006-01-02"), Time ("15:04:05"),
Channel (=PaymentCode), Hash (SHA1(PaymentCode+OrderNo) — อยู่ใน body ไม่ใช่ signature),
Withdraw (""), Deposit (amount), BankNo (=Username), BankCode (=PaymentCode),
Status (0 เสมอ), Detail (=PaymentCode), OrderNo, balance, amount_cost, profit,
bank_name, bank_number, bank_code, is_approve_bank,
bypass_notify_firebase (que_payment ใส่ true)
```

### Withdraw — `que_payment/models/automation_retry_callback.go:11-23` (3rd: `controller/withdraw.go:631-653`)
```
orderNo, transaction_id, status, comment, service, balance, cost_pay,
payment_code, cost_withdraw, cost_fix_withdraw, bypass_notify_firebase (que เท่านั้น)
```

### Payout — `que_payment/models/automation_retry_callback.go:25-42`
```
datetime, amount, bank_code, bank_name (=PaymentCode), bank_number (=paymentcode ตัวเล็ก),
comment, operator_name ("AUTO PAYMENT"), status, balance, cost_pay, payment_code,
orderNo, service, cost_withdraw, cost_fix_withdraw, bypass_notify_firebase
```

## Security & Delivery

- 🔴 **ไม่มี signature ใดๆ** — POST เปล่าๆ header เดียว `Content-Type: application/json` (`3rd-payment/service/callback/main.go:271`, `que_payment/services/callback/main.go:26`) — field `Hash` ใน deposit payload เป็นข้อมูล ไม่ใช่ลายเซ็น → rewrite ต้องเพิ่ม HMAC signing (design goal #4)
- **Success criteria**: merchant ตอบ `{code: 0}` หรือ `{msg: "Success"}` (case-insensitive ฝั่ง 3rd) — `service/callback/main.go:49` — HTTP status ไม่เช็คตรงๆ
- Timeout: 50s (3rd-payment) / 60s (que_payment) — ไม่เท่ากัน ⚠️

## Retry (delivery state ใน `callback_status`)

field `callback_status` บน `qr_payment` / `withdraw_statement`: **3 = ยังไม่ส่งสำเร็จ (ค่าเริ่มต้น), 1 = ส่งสำเร็จ**

Cronjob `AutomationRetryCallbackStatus` (`que_payment/cronjob/automation_retry_callback_status.go:34`) ทุก **30 นาที**:
- Deposit: query `callback_status != 1 AND status=1 AND is_success=1 AND datetime ใน 3 ชม.ล่าสุด` — ข้ามรายการอายุ < 10 นาที — batch 50 — Redis lock ต่อ order 10 นาที — sleep 10s ระหว่างรายการ (`:77-142`)
- Withdraw: query `callback_status != 1 AND status != 0 AND datetime ใน 6 ชม.ล่าสุด` — เงื่อนไขพิเศษ status 15/16 (`:144-167`, `:372-450`)
- **OfficeAPI URL ปัจจุบันดึงจาก record ล่าสุดที่ `callback_status=1`** ของ (payment_code, service) คู่นั้น — กลไกที่ทำให้ merchant เปลี่ยน URL ได้เอง ⚠️ subtle มาก ต้องคงพฤติกรรมหรือออกแบบ explicit registration แทน
- Manual retry: `POST /api/v2/retry-payment-sendffice` (`controller/deposit.go:4380`) — รับ `{payment_code, date}` ไล่ retry ทั้งวันนั้น
- 🔴 **ไม่มี max attempts / ไม่มี dead-letter** — retry ไปเรื่อยๆ จนกว่าจะสำเร็จหรือหลุดหน้าต่างเวลา (3/6 ชม.) แล้วเงียบหาย → rewrite ต้องมี DLQ ที่มองเห็นได้ (design goal #4)

## Rewrite Notes

- 🟢 Keep: retry cronjob concept, callback_status flag, merchant success-criteria (`code:0|msg:Success`)
- 🔴 Drop/Fix: payload เป็น bson.M ไร้ struct, ไม่มี signature, ไม่มี DLQ, OfficeAPI ดึงจาก record เก่า
- 🟡 Reshape: payload deposit/withdraw/payout 3 shape → unified event + type field (ต้องคุยกับ merchant ก่อน — เป็น breaking change)
