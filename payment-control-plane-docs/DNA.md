---
project: payment-control-plane
status: ready
last_reviewed: 2026-06-10
source_repos:
  - 3rd-payment
  - que_payment
deep_dive_docs: ./
---

# REWRITE DNA — `payment-control-plane`

> เอกสารแม่ — อ่านไฟล์นี้ก่อนทุกไฟล์ แล้วตามลิงก์ลงรายละเอียด (สร้างจาก `templates/rewrite-dna.md`)

## TL;DR — อ่าน 5 นาที

ระบบนี้คือ payment gateway ที่เป็นตัวกลางระหว่างเว็บ/แอปของลูกค้าเรา (เรียกว่า merchant) กับผู้ให้บริการรับ-จ่ายเงินภายนอกประมาณ 67 เจ้า (เรียกว่า provider เช่น maanpay, peer2pay) สิ่งที่ไหลผ่านระบบคือเงินจริง — ลูกค้าปลายทางฝากเงินผ่าน QR แล้วระบบต้องเอายอดไปเข้ากระเป๋า merchant ให้ถูกต้อง และ merchant สั่งถอนเงินออกไปเข้าบัญชีธนาคารปลายทาง ถ้าระบบพัง เงินจะหาย ซ้ำ หรือค้าง และ merchant ทุกเจ้าเดือดร้อนทันที

หัวใจของระบบมี 2 จังหวะ: **ฝาก (payin)** — merchant ขอ QR ลูกค้าโอน แล้ว provider ยิง callback กลับมาบอกว่าเงินเข้าแล้ว ระบบจึงบวกยอดเข้ากระเป๋าและแจ้งหลังบ้าน merchant และ **ถอน (payout/withdraw)** — merchant สั่งถอนแบบสองขั้น (สร้างรายการก่อน แล้วยืนยัน) ระบบหักยอดแล้วสั่ง provider โอนเงินจริง โค้ดส่วนนี้ถูกเขียนซ้ำ 2 ชุดใน 2 repo (`3rd-payment` รับ HTTP, `que_payment` ทำงานเบื้องหลังผ่าน queue) ซึ่งคือเหตุผลหลักของการ rewrite

ของที่ห้ามพังตอน rewrite คือ **contract กับโลกภายนอก 2 ฝั่ง**: รูปแบบ request/response ของ API ที่ merchant เรียกอยู่ (เปลี่ยน field เดียว = ระบบ merchant พัง) และรูปแบบ webhook ที่เรายิงไปแจ้งหลังบ้าน merchant — ทั้งสองมี schema บันทึกไว้แล้วใน `04-integrations/` นอกจากนี้กลไกกันรายการซ้ำ (unique index บน `hash`) และกติกาธุรกิจแฝง เช่น "ฝากซ้ำได้ order เดิมคืน" ต้องคงไว้ทั้งหมด

ของอันตรายที่สุดที่เจอตอนแกะ: (1) ระบบคิวสำรองที่ทุกคนเชื่อว่ามีจริงๆ แล้ว**ไม่มี** — ตัว poll สำรองแค่โยนงานกลับเข้า RabbitMQ ถ้า RabbitMQ ล่มทุกอย่างหยุด (2) ฝั่ง worker **ไม่มี distributed lock** ตอนแก้ยอดเงิน (โค้ด lock ถูก comment ทิ้ง) (3) callback จาก provider ราว 20 เจ้าไม่ถูกตรวจลายเซ็น และ webhook ที่เรายิงออกไม่มีลายเซ็นเลย (4) มีค่า status ปริศนา 2 ค่าที่ไม่มีโค้ดไหนใน 2 repo เป็นคนตั้ง — อาจมีระบบอื่นแอบเขียน database อยู่

ก่อนเริ่ม rewrite ต้องรอคำตอบ 9 ข้อจาก owner (อยู่ท้าย `REWRITE-PROMPT.md`) — ตัวชี้ขาดคือเรื่องฐานข้อมูล (อยู่ Mongo ต่อหรือย้าย Postgres), กลยุทธ์ queue, และเรื่อง lock ฝั่ง worker ว่าที่หายไปคือตั้งใจหรือ bug

## 0. Extraction Checklist

- [x] 1-11 ครบ และ verify กับโค้ดจริงแล้ว (audit 2026-06-10)
- [ ] 12. Open questions — **รอคำตอบ owner** (Q1-Q9 ใน `REWRITE-PROMPT.md`)
- [ ] 13. Verification plan — โครงมีแล้ว ต้องเติม NFR จาก owner

## 1. Project Identity

- **ทำอะไรให้ใคร:** payment gateway ตัวกลางระหว่าง merchant (เว็บ/แอป) กับ payment provider ภายนอก ~67 เจ้า — รับฝาก (payin/QR), สั่งถอน (payout), กระทบยอด, แจ้งผลกลับ merchant
- **อะไรไหลผ่าน:** **เงินจริง** — บั๊กแปลว่าเงินหาย/ซ้ำ ระดับความเสียหายสูงสุด
- **Source repos:**

| Repo | บทบาท | Tech | Entry points |
|---|---|---|---|
| `3rd-payment` | HTTP front door + cronjob | Go, Gin :8081, Mongo, Redis, RabbitMQ(pub) | `app.go:StartServ/StartCronjob` |
| `que_payment` | async worker + 3 cronjob modes | Go, RabbitMQ(consume), Mongo | `app.go:StartServ` switch ด้วย env `TYPE`/`PAYMENT_NAME` |

- **Scale/NFR:** ⚠️ TODO: verify with @owner (Q7-Q8)

## 2. Ecosystem Map

```
Merchant ──HTTP──▶ 3rd-payment ──MongoDB write + AMQP publish──▶ que_payment ──▶ Provider API
Provider ─callback─▶ 3rd-payment                                  que_payment ──webhook──▶ Merchant office
        ทั้งคู่: MongoDB (shared 5 collections!), Telegram alerts, monitor_payment event log
```

| ระบบ | ความสัมพันธ์ | Contract | rewrite แตะได้ไหม |
|---|---|---|---|
| Merchant systems | upstream + downstream | `04-integrations/merchant-api.md`, `office-webhook.md` | **ห้ามพัง wire format** |
| Provider ~67 เจ้า | downstream | `02-features/provider-matrix.md` | ตาม spec แต่ละเจ้า — แตะไม่ได้ |
| ENCRYPTION_ENDPOINT (Node.js) | เราพึ่ง (cloudpay, visapay, peer2pay, bitpayz) | ⚠️ ไม่มี doc — ใครดูแล? | ต้องตัดสินใจ inline/คงไว้ |
| Telegram (error + business bot) | เราพึ่ง | `bot/webhook` + notifier | อิสระ |
| ระบบปริศนาที่อาจเขียน DB | ⚠️ landmine #1 | — | ถาม owner ก่อน |

## 3. External Contracts (ห้ามพัง)

| Contract | ทิศทาง | Schema | ความเข้มงวด |
|---|---|---|---|
| Merchant API (`/payin`, `/create-withdraw`, `/confirm-withdraw`, statements, ...) | in | `04-integrations/merchant-api.md` | wire-compatible ต่อ endpoint |
| Office webhook (deposit/withdraw/payout payload) | out | `04-integrations/office-webhook.md` | wire-compatible (unsigned bson.M — แก้ได้แต่ต้อง coordinate) |
| Provider callbacks ~67 แบบ | in | `02-features/provider-matrix.md` | ตาม provider กำหนด |
| Queue message `ReqQue`/`ResQue` | internal | `01-source-repos/que_payment/README.md` | เปลี่ยนได้ถ้า migrate พร้อมกัน |
| MongoDB shared collections | internal-shared | `02-features/collection-ownership.md` | ต้องมี migration plan |

## 4. Core Business Capabilities (ห้ามขาด)

| Capability | Entry | Business rule แฝง | ระดับ |
|---|---|---|---|
| Payin (QR/bank) | `3rd/controller/deposit.go:103` | duplicate (code,service,user,amount) → คืน order เดิม `new_order:false`; lock 35s | **core** |
| Inbound callback + settle | `3rd/controller/callback.go:112` → que `DepositAgentV2` | slip fail → status 99 รอ office confirm; ตอบ 200 เสมอกัน provider retry | **core** |
| Payout/Withdraw two-phase | `3rd/controller/withdraw.go:38,308` | create→confirm แยกจังหวะ; hash ห้ามซ้ำ**ภายในวัน**; PAYOUT มีวงเงิน `CanWithdraw` | **core** |
| Balance / bank summary | `3rd/controller/Balance.go:74` | `UseDirectBalance` ข้าม ledger ไปถาม provider ตรง | **core** |
| Query/Reconcile (`check/order`) | `3rd/controller/transaction.go:494` | ปรับ state ย้อนหลังได้ | **core** |
| Outbound webhook + retry | que cronjob `automation_retry_callback_status.go:34` | OfficeAPI URL ดึงจาก record ล่าสุดที่ส่งสำเร็จ (merchant เปลี่ยน URL เองได้โดยปริยาย!) | **core** |
| Manual ops (confirm/cancel deposit 99) | `3rd/controller/main.go` + routes office group | ทางออกเดียวของเงินตกค้าง | **core** |
| Report/statement V1+V2 | `3rd/controller/main.go:747-857` | V1 ยัง live — audit caller ก่อนตัด | supporting |
| Corepay user mgmt, dashboard, business bot | routes กลุ่มเฉพาะ | ⚠️ Q3/Q4 — ใครใช้ | confirm-before-drop |

## 5. State & Invariants

- **State machines:** deposit `0→1 | 0→99→1` · withdraw `3→2|5→16→15→1` + fail `12/13/4` · delivery `callback_status 3→1` — ตาราง+setter ครบใน `02-features/order-state-machine.md`
- **Invariants:** ทุก `$inc` บน `bank_summary` มี `logs_inc_payment` คู่; status 1 = final เช็คก่อน mutate เสมอ; 13 ต้องคืนเงินเท่าที่ 16 หัก
- **Idempotency:** unique sparse index `hash` (qr_payment/withdraw_statement/adjust_statement — `3rd/repository/main.go:99-231`) + final-state check + thorlock (3rd เท่านั้น ⚠️)

## 6. Data Ownership

19 collections — รายละเอียด `02-features/collection-ownership.md` · **shared-write 5 ตัว** (bank_summary หนักสุด — ledger เงินจริงถูก `$inc` จาก 2 repo โดย que ไม่มี lock) · dead: `qr_payment_confirm` · index `hash` ×3 ห้ามหาย

## 7. Variation Points

จาก matrix 65 provider (`02-features/provider-matrix.md`):

| Variation | ค่าที่พบ | นัยต่อ design |
|---|---|---|
| Auth ขาออก | HMAC-SHA256 (~20), MD5-concat (~14), API-key เปล่า (~8), Bearer/login (~6), JWT/RSA/AES/SHA512 (เดี่ยวๆ) | `Signer` strategy ~6 ตัวครอบ ~90% |
| Callback verify ขาเข้า | sig / token / **none ~20 ตัว** | บังคับ verify + จัด tier ความเสี่ยง |
| Callback style | split (ส่วนใหญ่) / one-link / URL พิเศษ (hengpay, umpay, xpay, peer2pay) | 1 URL + dispatch by payload |
| Time format | unix, unix-ms, float, RFC3339, local string | per-provider codec |
| Error shape | code-field สารพัดเงื่อนไข / HTTP-status | normalize เป็น enum กลาง |
| Token lifecycle | บางเจ้า login+cache Redis | adapter ต้องมี state ได้ |

## 8. Cross-Cutting Patterns (เก็บ)

OTel ผ่าน AMQP headers (`amqpcarrier`) · Telegram severity 1-4 + dedup 5m · batch event writer 1s/1000 (`observability/mongodb/writer.go`) · error classifier · shutdown manager · per-provider queue isolation · comment_timeline audit · compensating reverse

## 9-10. Non-Negotiables / Kill List

แยกไฟล์: `05-rewrite/non-negotiables.md` + `05-rewrite/kill-list.md`

## 11. Landmines & Mysteries ⚠️

| สิ่งที่เจอ | Evidence | ต้องทำอะไร |
|---|---|---|
| withdraw status 2/5 มี guard แต่ไม่มี setter ใน 2 repo | `que/controllers/withdraw.go:100,359` | มีระบบอื่นเขียน DB? ถาม owner |
| Hybrid queue ไม่ standalone — poll แค่ re-publish เข้า MQ | `quepub/cornjob.go:202` | Rabbit ล่ม = หยุดหมด → ชี้ขาด Q6 |
| Worker ไม่มี distributed lock (comment ทิ้ง) | `que/services/bigpay/main.go:69` | Q9 |
| maanpay callback verify ถูก comment ทิ้ง | `maanpay/func.go:56-59` | ตั้งใจ? |
| Wallet lock `IsActive` ไร้ TTL | `quepub/cornjob.go:181-198` | ค้างต้องแก้มือ |
| Deposit ไม่มี expiry — callback ไม่มา = ค้าง 0 ตลอด | state machine doc | เพิ่ม expiry |
| Indo config path ข้าม auth | `3rd/controller/main.go:3070` | ตั้งใจ? |
| Sleep 5s รอ replication | `que/controllers/deposit.go:385` | แก้ด้วย write concern |

## 12. Open Questions → Owner

Q1-Q9 อยู่ใน `REWRITE-PROMPT.md` (USER PRE-ANSWERS) — **ต้องตอบครบก่อนเริ่ม** ตัวชี้ขาด design: Q5 (Mongo/Postgres), Q6 (queue), Q9 (locking)

## 13. Verification Plan

- **Contract tests:** golden request/response จาก `04-integrations/merchant-api.md` ต่อ endpoint — รันกับระบบใหม่ก่อน cutover
- **Shadow run:** ระบบใหม่รับ traffic คู่แบบ no-side-effect ต่อ provider ที่ migrate (REWRITE-PROMPT Phase 4)
- **Reconciliation:** เช็ค invariant section 5 อัตโนมัติ (ledger vs audit log) ระหว่าง shadow
- **Gate per provider:** parity N วัน + ไม่มี sev≥3 → cutover → ลบ code path เก่า

## 14. Deep-Dive Docs

| หัวข้อ | ไฟล์ |
|---|---|
| ภาพรวม + boundary | `RECAP.md`, `00-overview/system-context.md` |
| Surface ดิบต่อ repo | `01-source-repos/{3rd-payment,que_payment}/README.md` |
| Provider matrix / config / state / data | `02-features/*.md` |
| Workflows 3 เส้นหลัก | `03-workflows/*.md` |
| Contracts ห้ามพัง | `04-integrations/*.md` |
| เป้าหมาย rewrite | `05-rewrite/*.md` |
| Prompt สั่ง AI | `REWRITE-PROMPT.md` |
