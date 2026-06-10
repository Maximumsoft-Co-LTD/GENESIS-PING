---
last_reviewed: 2026-06-10
---

# 02-features

Cross-cutting DNA — pattern ที่ใช้ซ้ำข้าม provider 60+ ตัว

ใช้ template: `../templates/feature-doc.md`

## Feature ที่ควรเขียนก่อน (เรียงตาม priority)

1. `payin.md` — รับเงินเข้า (ครอบคลุม ~95% ของ provider)
2. `payout.md` — โอนเงินออก
3. `callback-inbound.md` — รับ webhook จาก provider
4. `callback-outbound.md` — ส่ง webhook ไปยัง merchant office
5. `idempotency.md` — pattern กลางสำหรับทุก state-mutating op
6. `distributed-lock.md` — thorlock pattern (⚠️ ใช้จริงเฉพาะ 3rd-payment — ฝั่ง que_payment ไม่มี lock)
7. `slip-verification.md` — verifySlip + first2pay
8. `qr-generation.md` — genQrImage
9. `balance-check.md` — balance + bank_summary
10. `reconciliation.md` — recheck order, retry callback
