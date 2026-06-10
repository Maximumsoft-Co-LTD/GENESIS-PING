---
project: payment-control-plane
status: extracting
last_reviewed: 2026-06-10
source_repos:
  - 3rd-payment   # Go + Gin HTTP API + provider integrations (60+ payment providers)
  - que_payment   # Go RabbitMQ consumer + background cronjobs
deep_dive_docs: N/A — แกะจาก zero context ในรอบนี้ (ของจริงควรแตกไฟล์ย่อยตาม note ในแต่ละ section)
---

# REWRITE DNA — `payment-control-plane`

> **นิยาม:** เอกสารเดียวที่ AI หรือ dev ที่ไม่เคยเห็นระบบ อ่านแล้วรู้ว่า ระบบนี้คืออะไร, หัวใจที่ห้ามหาย, อะไรห้ามพัง, อะไรตั้งใจทิ้ง — เพียงพอที่จะ rewrite โดยไม่ทำธุรกิจเสียหาย
>
> **สถานะความเชื่อมั่น:** เอกสารนี้แกะจากการอ่านโค้ดจริง (entry points, routes, models, repository, consumer, cronjobs, provider auth). ส่วนที่เป็น business intent / NFR / ค่า magic number ที่หาคนกำหนดไม่เจอ ถูกลงไว้ใน section 11–12 ไม่เดาแทน owner

---

## บันทึกการทดลอง: ผล cross-check กับชุด docs เดิม (2026-06-10)

ไฟล์นี้คือ**ผลการทดสอบ template + prompt** ใน `templates/rewrite-dna.md` — สร้างโดย AI ที่ zero-context (ห้ามอ่าน docs เดิมทั้งหมด อ่านได้แค่ template) แล้วเอามาเทียบกับชุด canonical (`DNA.md` + deep-dive docs ที่สร้างจากหลาย session, 12+ agents) ผลที่ได้:

### ข้อผิดพลาดของ docs เดิมที่ไฟล์นี้จับได้ (verify กับโค้ดแล้วทุกข้อ)

| # | Docs เดิมเขียนว่า | ความจริงจากโค้ด | Evidence |
|---|---|---|---|
| 1 | timeout ถอนค้าง → status 4 อยู่ที่ `cronjob/_retry.go:76` window 30 วัน (`02-features/order-state-machine.md`) | `_retry.go` ขึ้นต้น `_` = **Go ไม่ compile (dead code)** — ตัวจริงที่ทำงานคือ `RetryOrderCallbackAgain` ใน `cronjob/retry-order.go` (ยกเลิกเฉพาะ status 16) | `que_payment/cronjob/retry-order.go:74-86` |
| 2 | wallet lock `bank_summary.is_active` "ไร้ TTL — ค้างต้องแก้มือ" (`03-workflows/payout-end-to-end.md`, `05-rewrite/kill-list.md`) | มี cronjob `BANK_SUMMARY_RECOVER` ปลดล็อกกระเป๋า/คิวที่ค้างเกิน **15 นาที**อัตโนมัติ | `que_payment/cronjob/recover_bank_summary.go:101-113` |
| 3 | duplicate payin เช็คจาก (payment_code, service, username, amount) โดยไม่ระบุ window (`04-integrations/merchant-api.md`) | window แค่ **5 นาที** — พ้นช่วงนี้สร้าง order ใหม่ได้ | `3rd-payment/repository/v2.go:248` (`GetDepositStatementOrderOldV2`) |

### Landmines ใหม่ที่ docs เดิมไม่เคยบันทึก (spot-check ยืนยันแล้ว)

- 🔴 **Hash bug ใน production**: `HashWithdraw`/`HashPayOut` สร้าง `hashSha1Pm` แต่เรียก `Sum` จาก `hashSha1` ตัวแรก → ค่า hashPm ที่ควรรวม paymentCode ไม่เคยถูกคำนวณถูก (`3rd-payment/service/hashlayout/main.go:50,71` — ยืนยันด้วยการอ่านโค้ดตรง)
- 🔴 **Queue routing 2 ชื่อ**: 3rd-payment publish ไป `QUE_MERCHANT_<merchant_code>` ได้ (`3rd-payment/service/rabbit/que-payment.go:35`) แต่ consumer declare เฉพาะ `QUE_PAYMENT_*` (`que_payment/rabbitmqpub/amqp.go:62,233`) — ใครกิน QUE_MERCHANT? ⚠️ TODO: verify with @owner
- field `merchant_code` อยู่ในฝั่ง publish แต่ struct `ReqQue` ฝั่ง consume ไม่มี
- 3rd-payment มีจุด `$inc` ledger ที่ไม่เขียน `logs_inc_payment` (`3rd-payment/repository/v2.go:1794`) — ละเมิด invariant ledger+audit
- `monitor_payment` มี **TTL index 7 วัน** (`observability/mongodb/writer.go:18,66`) — event log หายเองหลัง 7 วัน
- การ publish เข้า queue เป็น **synchronous RPC** (publish + รอ reply, timeout 120s — `3rd-payment/service/rabbit/que-payment.go:20,78-209`) ไม่ใช่ fire-and-forget

### บทเรียน

1. **Template + prompt ใช้งานได้จริง** — AI zero-context ผลิต DNA คุณภาพสูงใน ~19 นาที และซื่อสัตย์ต่อกติกา (ไม่ติ๊ก checklist ข้อที่ไม่ครบ)
2. **การแกะอิสระซ้ำ (independent re-extraction) คือเครื่อง cross-check ที่ดีที่สุด** — มุมมองใหม่เจอของที่รอบแรกตาบอด และจับ error ของรอบแรกได้ — ควรทำกับโปรเจคสำคัญทุกตัวก่อน hand-off
3. จุดอ่อนของ single-shot: ความครอบคลุม (ไม่มี per-provider matrix, route ไม่ครบ) — งาน fan-out ยังต้องทำแยก

> สถานะไฟล์นี้: เก็บไว้เป็นบันทึกการทดลอง + แหล่ง findings ที่รอ merge เข้าชุด canonical — อย่าใช้แทน `DNA.md`

---

## TL;DR — อ่าน 5 นาที (สำหรับ dev ที่เพิ่งเข้าทีม)

ระบบนี้คือ **payment integration hub** (เกตเวย์รวมช่องทางชำระเงิน) ที่นั่งกลางระหว่าง "เว็บลูกค้า/office ของกลุ่ม" กับ **ผู้ให้บริการชำระเงินภายนอกกว่า 60 ราย** (speedpay, wowpay, xpay, beepay, peer2pay ฯลฯ). หน้าที่หลักคือรับคำสั่ง **ฝาก (deposit/payin)** และ **ถอน (withdraw/payout)** จากเว็บ แล้วแปลงเป็น request ที่แต่ละ provider เข้าใจ (แต่ละเจ้า sign/auth ต่างกัน) พร้อมรับ callback (webhook) กลับมาปรับยอดและยิงผลกลับเข้า back-office. ถ้าระบบนี้พัง = ลูกค้าฝากเงินไม่เข้า/ถอนไม่ออก และยอดเงินใน ledger (`bank_summary`) อาจเพี้ยน — กระทบเงินจริงโดยตรง.

หัวใจของระบบมี 3 ก้อน: (1) **provider abstraction** — โค้ด ~60 โฟลเดอร์ใน `3rd-payment/controller/<provider>/` แต่ละตัวห่อ auth + request/response ของ provider นั้น, (2) **money ledger** — `bank_summary.current_balance` ที่ถูก `$inc` คู่กับ audit row ใน `logs_inc_payment` ทุกครั้ง, (3) **คิวถอนแบบ RPC ผ่าน RabbitMQ** — `3rd-payment` publish เข้า queue แล้ว block รอ reply จาก `que_payment` (ทำหน้าที่ประมวลผลถอน/refund/report จริง).

อะไรห้ามพังตอน rewrite: (a) **external HTTP contract** ของ deposit/withdraw/callback routes ที่เว็บลูกค้าและ 60 provider integrate อยู่ (เปลี่ยน path/field เดียว = breaking), (b) **โครงสร้าง MongoDB ที่สอง repo เขียนร่วมกัน** (`bank_summary`, `qr_payment`, `withdraw_statement`, `que_payment`), (c) **กลไก idempotency** ที่พึ่ง unique index บน field `hash` ของ qr_payment/withdraw_statement และ status guards.

ของอันตรายที่สุดที่เจอ: **มี code path v1 (legacy) กับ v2 อยู่คู่กันเต็มไปหมด** (DepositAgent vs DepositAgentV2, UpdateSummaryByService vs ...V2 ที่ signature ต่างกัน), **status code เป็นตัวเลขดิบ (0,1,2,3,4,5,12,13,15,16,99) ที่ไม่มี enum / ไม่มี doc กลาง**, และ **bug แฝงใน HashWithdraw/HashPayOut ที่คืน hashPm จาก hash ตัวผิด** (ดู section 11). ก่อน rewrite ต้องได้คำตอบจาก owner เรื่องความหมายที่แท้จริงของ status แต่ละตัว, ledger เป็น source of truth จริงไหม หรือ provider เป็น, และ v1 paths ยัง live อยู่หรือไม่ (section 12).

> เกณฑ์ผ่าน: dev อ่านจบเล่าต่อได้ว่า "เกตเวย์รวม payment 60 เจ้า, เงินอยู่ใน bank_summary, ถอนทำผ่าน RabbitMQ RPC, ระวัง v1/v2 ปนกันและ status ตัวเลขดิบ"

## 0. Extraction Checklist (สถานะความครบของ DNA นี้)

- [x] 1. Identity — ภาพรวม + บทบาทแต่ละ repo
- [x] 2. Ecosystem — ระบบรอบข้าง + ใครพึ่งเรา/เราพึ่งใคร
- [x] 3. External contracts — โครงหลัก verify แล้ว (รายละเอียด field ของแต่ละ provider ยังไม่ครบ — ดู note)
- [x] 4. Core capabilities — function ที่เป็นธุรกิจแท้ แยกจากโค้ดประกอบ
- [x] 5. State machines + invariants — status หลัก verify แล้ว, ความหมายบางค่ายัง TODO
- [x] 6. Data ownership — collection หลัก + shared-write verify แล้ว
- [x] 7. Variation points — auth matrix 60+ provider (verify spot-check แล้ว)
- [x] 8. Cross-cutting patterns
- [x] 9-10. Non-negotiables + Kill list
- [x] 11. Landmines
- [x] 12. Open questions
- [ ] 13. Verification plan — ร่างไว้แล้ว แต่ยังต้อง owner ยืนยัน parity baseline ก่อนถือว่าครบ

---

## 1. Project Identity (ภาพรวม)

ระบบนี้คือ control plane รวมการชำระเงิน — รับ deposit/withdraw จากเว็บลูกค้า แปลงไปยัง payment provider แต่ละเจ้า และ reconcile ยอดกลับ. แบ่งเป็น 2 service ที่แชร์ MongoDB เดียวกัน.

- **ระบบนี้ทำอะไร ให้ใคร:** เกตเวย์/ตัวกลางชำระเงิน ให้เว็บ/office ในกลุ่มเรียกใช้เพื่อเชื่อมต่อ payment provider ภายนอก 60+ ราย โดยไม่ต้อง integrate แต่ละเจ้าเอง
- **อะไรไหลผ่านระบบ:** **เงินจริง** (ฝาก-ถอน) + ข้อมูลบัญชีธนาคารลูกค้า + เครดิต/ยอดคงเหลือ. ความเสียหายถ้าพัง = เงินเข้า/ออกผิด, ยอด ledger เพี้ยน, ฝาก/ถอนค้าง
- **Source repos:**

| Repo | บทบาท | Tech | Entry points |
|---|---|---|---|
| 3rd-payment | HTTP API (รับ deposit/withdraw/callback จากเว็บและ provider) + ตัว integrate provider 60+ ตัว + telegram bot + cronjob เก็บ statement | Go, Gin, MongoDB, Redis (lock), OTel | `3rd-payment/_cmd/main.go:29` → `app.StartServ` (`3rd-payment/app.go:33`); routes ที่ `3rd-payment/routes/main.go:14`, `routes/bank-transfer-gateway.go:14`, `routes/call-by-payment.go:33` |
| que_payment | RabbitMQ consumer ประมวลผลถอน/refund/report จริง + cronjob (recover ledger, retry callback, update redis) | Go, RabbitMQ (streadway/amqp), MongoDB, Redis, Sentry, OTel | `que_payment/_cmd/main.go:27` → `app.StartServ` (`que_payment/app.go:22`); consumer `que_payment/rabbitmqpub/amqp.go:48`; cronjobs `que_payment/cronjob/*.go` |

- **Mode switching (que_payment):** เลือกพฤติกรรมจาก env `TYPE` — `QUE_RABBIT` = รัน consumer; `QUE_CRONJOB` = รัน cronjob โดยแยกย่อยตาม `PAYMENT_NAME` (`BANK_SUMMARY_RECOVER`, `AUTOMATION_RETRY_CALLBACK_STATUS`, default = updateRedis+retryOrder+queuePoller) — `que_payment/app.go:95-167`
- **Scale / NFR:** ⚠️ TODO: verify with @owner — throughput/latency/uptime หาจากโค้ดไม่ได้. เห็นเพียง consumer prefetch `Qos(1,...)` (`que_payment/rabbitmqpub/amqp.go:74`) และ worker pool 10 ตัวใน queue poller (`que_payment/quepub/cornjob.go:66`); RPC timeout 120s (`3rd-payment/service/rabbit/que-payment.go:20`)

## 2. Ecosystem Map (repo และระบบที่เกี่ยวข้อง)

ภาพรวมการคุยกัน: เว็บลูกค้า/office → 3rd-payment (HTTP) → (สำหรับถอน/refund/report) RabbitMQ RPC → que_payment → provider ภายนอก; provider → callback HTTP → 3rd-payment; ทั้งสอง service เขียน MongoDB ก้อนเดียวกัน.

```
[เว็บลูกค้า / Back-office]
        │  HTTP (deposit/withdraw/balance/statement)
        ▼
┌──────────────────────┐      callback (webhook)      ┌─────────────────────┐
│     3rd-payment      │◀─────────────────────────────│ Payment Providers   │
│   (Gin HTTP :8081)   │──── deposit request ────────▶│   60+ ราย           │
└──────────┬───────────┘                              └─────────────────────┘
           │ AMQP RPC (publish + รอ reply บน exclusive queue)
           │ queue: QUE_PAYMENT_<PAYMENT_NAME> หรือ QUE_MERCHANT_<MERCHANT_CODE>
           ▼
┌──────────────────────┐      provider API            ┌─────────────────────┐
│     que_payment      │─────(withdraw/refund)───────▶│ Payment Providers   │
│ (RabbitMQ consumer + │                              └─────────────────────┘
│  cronjobs)           │       callback ผลกลับ        ┌─────────────────────┐
└──────────┬───────────┘──────(withdraw/deposit)─────▶│ Back-office API     │
           │                                          │ /api/WithdrawCallback│
           │ shared MongoDB                           │ /api/Payment/...     │
           ▼                                          └─────────────────────┘
   [MongoDB: bank_summary, qr_payment, withdraw_statement, que_payment, service_qrpayment, ...]
   [Redis: distributed lock (thorlock), payment_bank_support cache]
   [Telegram: business bot (3rd-payment) + error/oncall notifier (ทั้งสอง)]
```

| ระบบ/Repo | ความสัมพันธ์ | Contract | เจ้าของ | rewrite แตะได้ไหม |
|---|---|---|---|---|
| เว็บลูกค้า / back-office | upstream (เรียกเรา) + downstream (เรา callback กลับ) | HTTP routes ใน `3rd-payment/routes/main.go`; callback ออกที่ `que_payment/services/callback/main.go:33` (path `api/WithdrawCallback`, `api/Payment/PaymentCreateStatement/<service>`, `api/Payment/CallBackWithdrawAmount/<service>`) | ⚠️ TODO: ทีม web/office | ต้อง coordinate (breaking ถ้าเปลี่ยน) |
| Payment providers (60+) | downstream (เราเรียก) + upstream (callback เข้า) | ต่อ provider — โค้ดที่ `3rd-payment/controller/<provider>/` และ `que_payment/services/<provider>/` | external (ผู้ให้บริการ) | read-only (contract ของเขา) |
| MongoDB | shared-db (สอง repo เขียนร่วม) | collections ใน section 6 | internal | ต้อง coordinate (เปลี่ยน schema กระทบทั้งคู่) |
| RabbitMQ | shared-infra (3rd-payment publish, que_payment consume) | queue name + message struct (section 3); URL มาจาก `service_qrpayment.rabbitmq_url` | internal | อิสระ (แต่ต้องคง message contract) |
| Redis | shared-infra (distributed lock + cache) | thorlock (`3rd-payment/service/thorlock`), redis client (`que_payment/redis`) | internal | อิสระ |
| Telegram | downstream (แจ้งเตือน + bot) | business bot (`3rd-payment/app.go:40`), error notifier (`observability/telegram`) | internal | อิสระ |
| OTel collector | downstream (trace) | gRPC `OTEL_CONNECT` (`observability/init.go:13`); collector config `3rd-payment/config.yaml` | internal | อิสระ |
| Sentry | downstream (error) — เฉพาะ que_payment | `SENTRY_DSN` (`que_payment/_cmd/main.go:33`) | internal | อิสระ |

## 3. External Contracts (ห้ามพังเด็ดขาด)

ทุกอย่างที่คนนอกทีม integrate อยู่. inbound HTTP มาจากเว็บลูกค้าและ provider callback; outbound คือ callback ที่เรายิงกลับ office; internal-but-shared คือ AMQP message ระหว่างสอง repo.

| Contract | ทิศทาง | Consumer | Schema อยู่ที่ | ระดับความเข้มงวด |
|---|---|---|---|---|
| `POST /api/v2/payin` (deposit) | inbound | เว็บลูกค้า | req struct `ReqDeposit` (`3rd-payment/controller/model.go:11`), handler `controller.Deposit` (`3rd-payment/controller/deposit.go:103`), route `routes/main.go:53` | wire-compatible |
| `POST /api/v2/create-withdraw` + `/confirm-withdraw` | inbound | เว็บลูกค้า | `ReqCreateWithdraw` (`model.go:80`), `controller.CreateWithdraw` (`controller/withdraw.go:38`), `ConfirmWithdraw` (`withdraw.go:308`), route `routes/main.go:60-61` | wire-compatible |
| `POST /api/v2/callback-payin/:payment_code` (+ `/callback/qrStatus`, one-link variants) | inbound | payment provider (webhook) | `controller.CallbackDeposit` (`controller/callback.go:112`), route `routes/main.go:55-59`; ผ่าน IP whitelist middleware (`middleware/ipwhitelist.go:21`) | wire-compatible (ต่อ provider) |
| `POST /api/v2/callback-payout/:payment_code` (+ autopeer/umpay variants) | inbound | payment provider | `controller.CallbackWithdraw` (route `routes/main.go:64-68`) | wire-compatible |
| Response envelope (เว็บ) | outbound | เว็บลูกค้า | `helper.Resp` → `{code,msg,data,error}` โดย code 0/1 → HTTP 200 เสมอ (`3rd-payment/helper/resp.go:25`) | wire-compatible — **code semantics กลับด้านในบางที่** (ดู landmine) |
| Response envelope (provider callback) | outbound | provider | `helper.PaymentResp` ปรับรูปต่อ provider เช่น COREPAY → `{code,message}`, HENGPAY → `{responseCode,responseMesg}` (`helper/resp.go:40`) | wire-compatible ต่อ provider |
| Callback กลับ office (withdraw) | outbound | back-office | path `api/WithdrawCallback`, payload `{orderNo,status,comment,service,balance}` (`que_payment/quepub/cornjob.go:225`, `services/callback/main.go:41`) | wire-compatible |
| Callback กลับ office (deposit) | outbound | back-office | path `api/Payment/PaymentCreateStatement/<service>`, payload bson.M (`que_payment/controllers/deposit.go:494`) | wire-compatible |
| Callback กลับ office (เพิ่มทุนถอนจากฝาก) | outbound | back-office | path `api/Payment/CallBackWithdrawAmount/<service>` (`que_payment/services/callback/main.go:45`) | wire-compatible |
| AMQP message (deposit/withdraw/report) | internal-but-shared | que_payment | publish struct `{id,service,type,type_report,date,payment_code,merchant_code}` (`3rd-payment/service/rabbit/que-payment.go:114`); consume struct `ReqQue` (`que_payment/rabbitmqpub/amqp.go:33`); reply `ResQue{code,message}` | versioned-careful (สอง repo ต้องตรงกัน; field `merchant_code` มีฝั่ง publish แต่ `ReqQue` ฝั่ง consume **ไม่มี** — ดู landmine) |
| Generic provider proxy `ANY /call-pm/:paymentCode/*byPath` | inbound | ⚠️ TODO: ใครเรียก | `RouteCallByPayment` (`routes/call-by-payment.go:33`), forward ไป `service_qrpayment.host_smart_pay` (`controller/call-by-payment.go:51`) | ⚠️ ต้อง verify ใครใช้ก่อนแตะ |

> หมายเหตุ: route ทั้งหมดมี **80+ endpoint** ใน `routes/main.go` เพียงไฟล์เดียว (deposit, withdraw, balance, statement, report, dashboard, corepay member, hengpay, peer2pay, bot). ของจริงควรแตกเป็นไฟล์ contract แยกต่อกลุ่ม. ตารางนี้ครอบเฉพาะ contract money-path หลัก. รายละเอียด request/response **ต่อ provider** (60+ ราย) ยังไม่ได้ enumerate — เป็นงาน fan-out ที่ต้องแตกไฟล์ matrix แยก.

## 4. Core Business Capabilities (หน้าที่สำคัญห้ามขาด)

แยกธุรกิจแท้ (ฝาก-ถอน-ปรับยอด-reconcile) ออกจากโค้ดประกอบ (logging, tracing, telegram). ตารางนี้คือ capability ที่ถ้าขาด = rewrite ล้มเหลว.

| Capability | ทำอะไร (มุมธุรกิจ) | Entry points (file:line) | Business rule แฝง | ระดับ |
|---|---|---|---|---|
| Deposit / payin | สร้างรายการฝาก, gen QR/บัญชีจาก provider, รอ callback | `3rd-payment/controller/deposit.go:103` (`Deposit`); dispatch `SwitchPaymentDeposit` (`deposit.go:728`) | ถ้ามี order เดิม (same payment_code+service+username+amount ภายใน 5 นาที, is_success≠1) คืน order เดิม ไม่สร้างใหม่ (`repository/v2.go:246`); Redis lock key `paymentCode:username:amount` 35s กัน double (`deposit.go:177`) | core |
| Deposit callback → credit | รับ webhook provider, ปรับ ledger, ยิงกลับ office | `controller/callback.go:112` (`CallbackDeposit`) → `CallRabbitDeposit` (`callback.go:15060`) → consumer `que_payment/controllers/deposit.go:94` (`DepositAgentV2`) | credit เข้า ledger = `AmountCreate - (CostDeposit+CostWithdraw+Profit)` (`que_payment/controllers/deposit.go:348`); ถ้า update is_success ล้มเหลว → rollback ยอด (`deposit.go:401`); guard `is_success==1 || status==1` กันทำซ้ำ (`deposit.go:117`) | core |
| Withdraw / payout | สร้างรายการถอน, ตรวจ balance, ส่งคิวให้ provider จ่าย | สร้าง: `controller/withdraw.go:38` → InsertQue → poller `que_payment/quepub/cornjob.go:101` (`QueRunningV2`); ประมวลผล: `que_payment/controllers/withdraw.go:338` (`WithdrawAndPayOutV2`) → switch ต่อ provider (`withdraw.go:439`) | ตัดยอด ledger ก่อนยิง provider แล้ว refund ถ้าล้มเหลว (`que_payment/controllers/withdraw.go:260,278`); ทำได้เฉพาะ status que 2 หรือ 5 (`withdraw.go:359`); lock bank_summary ด้วย is_active flag ระหว่างประมวลผล (`quepub/cornjob.go:181`) | core |
| Money ledger ($inc + audit) | บวก/ลบยอดคงเหลือ + เขียน audit ทุกครั้ง | `que_payment/repository/paymentV2.go:136` (`UpdateSummaryByServiceV2`); v1 `payment.go:166`; 3rd-payment v2 `repository/v2.go:1794` | ทุก `$inc current_balance` ใน que_payment สร้าง `logs_inc_payment` คู่ (amount_before/after) — **แต่ 3rd-payment เวอร์ชันไม่เขียน log** (ดู landmine) | core |
| Refund (ถอนล้มเหลว/หมดอายุ) | คืนยอดเข้า ledger เมื่อถอนไม่สำเร็จ | consumer type `REFUND` → `que_payment/controllers/refund.go:11` (`RefundMoney`/V2); office refund `RefundWithdrawOffice` (`controllers/withdraw.go:6094`); cron retry-order ยกเลิก+refund (`cronjob/retry-order.go:74`) | refund ทำได้เฉพาะ withdraw status 15 (`refund.go:34`, `withdraw.go:6111`) | core |
| Adjust balance (manual) | office ปรับยอด ledger มือ | `3rd-payment/controller/adjustBalance.go:163` | idempotent ด้วย unique `hash` index บน adjust_statement (`repository/main.go:220`) | core |
| Provider config CRUD | สร้าง/แก้ config payment + test | `routes/main.go:37-44` (create/update/test config) | test payment สำเร็จ → set status 1, ล้มเหลว → status 0 (`deposit.go:601,613`) | supporting |
| Report aggregation | สรุปยอด deposit/withdraw รายวัน/เดือน | consumer type `REPORT_DATA_CURRENT/OLD` → `que_payment/controllers/report.go:21` (`UpdateReport`) | — | supporting |
| Balance / statement query | เว็บ/office ดูยอด, รายการ | `routes/main.go:88-99` (balance, deposit/withdraw/payout statement V1+V2) | — | supporting |
| Bank summary recover | ปลดล็อก bank_summary/queue ที่ค้าง >15 นาที | `que_payment/cronjob/recover_bank_summary.go:18` | unlock ถ้าไม่มี que status 2 ค้าง หรือ que เก่ากว่า 15 นาที (`recover_bank_summary.go:101,113`) | supporting (กัน deadlock ledger lock) |
| Telegram business bot | office สั่งงาน/ดู log ผ่าน telegram | `3rd-payment/app.go:40`, routes `routes/main.go:155-159`, `controller/botTelegram.go` | — | supporting |
| Bank transfer gateway | จัดการ bank gateway สำหรับถอน | `routes/bank-transfer-gateway.go:14`, `controller/banktransfer/` | AES key manage ถูก comment ทิ้ง (ดู landmine) | legacy-confirm-before-drop |
| Generic provider proxy | forward request ดิบไป provider host | `controller/call-by-payment.go:51` | — | legacy-confirm-before-drop |

## 5. State & Invariants

ระบบใช้ field `status` (int) บน `qr_payment` (ฝาก), `withdraw_statement` (ถอน), `que_payment` (คิว) เป็น state machine หลัก แต่ **ค่าทั้งหมดเป็นตัวเลขดิบ ไม่มี enum/const กลาง** — ความหมายต้องเดาจาก context การใช้งานและ comment ภาษาไทย. ตารางด้านล่างคือค่าที่ verify จากโค้ดได้; ค่าที่ยังคลุมเครือลง section 12.

**Withdraw statement status (`withdraw_statement.status`):**

| Status | ความหมาย (เท่าที่ verify ได้) | Set โดย (file:line) | Transitions |
|---|---|---|---|
| 3 | สร้างรายการถอนแล้ว รอ confirm | `controller/withdraw.go:585` (ConvertToCreateWithdraw set 3) | 3 → 2 (confirm, withdraw.go:392) |
| 2 | confirm แล้ว เข้าคิวรอประมวลผล | `controller/withdraw.go:392` | 2 → 16/15/12/13 |
| 5 | confirm รอบใหม่ (รายการเคย error) | `controller/withdraw.go:402` | 5 → ประมวลผล (consumer ยอมรับ status 2 หรือ 5: `que_payment/controllers/withdraw.go:359`) |
| 16 | กำลังดำเนินการถอน/ส่งรายการให้ provider | `que_payment/controllers/withdraw.go:184,267` | 16 → 15 (สำเร็จ) / 12,13 (fail) / 4 (cron ยกเลิกถ้าค้าง) |
| 15 | ส่งรายการถอนสำเร็จ รอ provider ยืนยัน (callback) | `que_payment/controllers/withdraw.go:216,271,306` | 15 → 1 (callback success: `callback.go:15174`) / refund ได้ |
| 1 | สำเร็จ (is_success=1) | `que_payment/controllers/withdraw.go:368`, `callback.go:15171` | terminal |
| 12 | ล้มเหลว/ยกเลิก (คืนยอดถ้าตัดไปแล้ว) | ทั่ว `que_payment/controllers/withdraw.go` (12 = "ไม่สามารถทำรายการ") | terminal |
| 13 | ล้มเหลวหลังเรียก provider (payment error) | `que_payment/controllers/withdraw.go:279,285,300` | terminal |
| 4 | หมดอายุ/ถูกยกเลิกโดย cron | `que_payment/cronjob/retry-order.go:75-86` (status 16 ค้าง 5ชม.-3วัน → 4) | terminal |

**Deposit (qr_payment) status:**

| Status | ความหมาย | Set โดย | Transitions |
|---|---|---|---|
| 0 | สร้างรายการฝาก รอเงินเข้า | `controller/deposit.go:453` (ConvertToCreateDeposit) | 0 → 1 / 99 |
| 1 | ฝากสำเร็จ (is_success=1) | `que_payment/repository/paymentV2.go:108` (UpdateIsSuccessDepositStatementV2) | terminal |
| 99 | รอ confirm manual (ฝากตกค้าง / ยอดไม่ตรง) | `controller/callback.go:212-213` (set 99 + update qr_payment); office confirm ที่ route `/confirm-bank-deposit` | 99 → 1 (office confirm) |
| `callback_status` 0/1/3 | สถานะ callback กลับ office (3 = ยังไม่ยิง, 1 = ยิงสำเร็จ) | `controller/deposit.go:463` set 3; `que_payment/repository/paymentV2.go:116` set 1 | — |

**Que payment status (`que_payment.status`):**

| Status | ความหมาย | Set โดย |
|---|---|---|
| 0 | สร้างคิว รอประมวลผล | `controller/withdraw.go:422` (InsertQue) |
| 2 | กำลังประมวลผล (lock) | `que_payment/repository/paymentV2.go:219` (UpdateQueByStatusV2 → 2), `quepub/cornjob.go:119` |
| 1 | ประมวลผลเสร็จ (จบ ไม่ว่า success/fail) | `que_payment/quepub/cornjob.go:212` |

**Invariants (สิ่งที่ต้องจริงเสมอ):**
- ทุก `$inc` บน `bank_summary.current_balance` **ในฝั่ง que_payment** ต้องมี `logs_inc_payment` audit row คู่กัน (`que_payment/repository/paymentV2.go:150-194`) — มี amount_before/after. ⚠️ **ฝั่ง 3rd-payment (`repository/v2.go:1794`) $inc โดยไม่เขียน log** = ละเมิด invariant นี้ (landmine).
- การถอนต้อง: lock `bank_summary.is_active=1` ก่อน → ตัดยอด → ยิง provider → ปลดล็อก (`quepub/cornjob.go:181-198`). ถ้า process ตาย ledger ค้าง lock จนกว่า cron recover (`recover_bank_summary.go`).
- ถอนตัดยอดก่อน ถ้า provider ล้มเหลว ต้อง refund คืน (`que_payment/controllers/withdraw.go:278`).

**Idempotency:**
- **Deposit & Withdraw:** unique sparse index บน field `hash` ของ `qr_payment`/`withdraw_statement` สร้าง on-the-fly ก่อน insert (`3rd-payment/repository/main.go:92-108,202-218`). hash = SHA1(date+service+username/bankNumber+amount[+paymentCode]) (`service/hashlayout/main.go:16,41`).
- **Adjust:** unique `hash` index บน `adjust_statement` (`repository/main.go:220`).
- **Deposit reuse:** order เดิมภายใน 5 นาทีถูกคืนแทนสร้างใหม่ (`repository/v2.go:246`).
- **Deposit credit guard:** `is_success==1 || status==1` → ปฏิเสธ (`que_payment/controllers/deposit.go:117`).
- **Redis lock:** deposit lock `paymentCode:username:amount` 35s (`deposit.go:177`); recover cron lock `recover_bank_summary_lock` (`recover_bank_summary.go:20`); retry-callback lock keys (`cronjob/automation_retry_callback_status.go:28`).

## 6. Data Ownership

ทั้งสอง repo เขียน MongoDB ก้อนเดียว (`DB_DBNAME`/`DB_ENDPOINT`, `db/connect.db.go:38` / `que_payment/db/conn.go:37`). หลาย collection เป็น **shared-write** = อันตรายสุดตอน rewrite. collection ส่วนใหญ่อ้างด้วย string literal ไม่มี const กลาง (ยกเว้นใน banktransfer).

| Store | Object (collection) | ใครเขียน | ใครอ่าน | ⚠️ shared-write? | Index/constraint ห้ามหาย |
|---|---|---|---|---|---|
| MongoDB | `service_qrpayment` | 3rd-payment (config CRUD) | ทั้งสอง (GetConfigPaymentV2) | เขียนฝั่ง 3rd เป็นหลัก | payment_code lookup |
| MongoDB | `qr_payment` (deposit statement) | ทั้งสอง | ทั้งสอง | ✅ ใช่ | **unique sparse `hash`** (idempotency); lookup payment_code+orderNo+service |
| MongoDB | `withdraw_statement` | ทั้งสอง | ทั้งสอง | ✅ ใช่ | **unique sparse `hash`**; lookup payment_code+orderNo+service |
| MongoDB | `bank_summary` (money ledger) | ทั้งสอง ($inc current_balance) | ทั้งสอง | ✅ ใช่ (เงินจริง) | filter payment_code+service+currency+exchange; field `is_active` (lock); `current_balance` |
| MongoDB | `que_payment` (withdraw queue) | 3rd (insert), que (update status) | que (poller) | ✅ ใช่ | status, payment_code, service |
| MongoDB | `logs_inc_payment` (ledger audit) | que_payment (+3rd บางจุด) | (audit) | append-only | — |
| MongoDB | `logs_que_payment` | que_payment | — | — | — |
| MongoDB | `logs_callback_payment` | 3rd-payment | que (GetLogCallbackPaymentByOrderNo) | shared-read | orderNo |
| MongoDB | `logs_callback_office` | que_payment | — | — | — |
| MongoDB | `adjust_statement` | 3rd-payment | 3rd | — | unique `hash` |
| MongoDB | `confirm_bank_deposit` | 3rd-payment | ทั้งสอง | — | — |
| MongoDB | `report_payment` | que_payment | 3rd (query report) | shared | — |
| MongoDB | `telegram_order_statement`, `telegram_connect_provider`, `bot_telegram` | 3rd-payment | 3rd | — | — |
| MongoDB | `monitor_payment` (observability events) | ทั้งสอง (mongodb.Writer) | (observability) | append | **TTL index 7 วัน บน event_time** (`observability/mongodb/writer.go:18,66`) |
| MongoDB | `service_bank_transfer_gateway` ฯลฯ | banktransfer | banktransfer | — | unique `hash` (`banktransfer/main.go:623`) |
| Redis | lock keys + `payment_bank_support` cache | ทั้งสอง | ทั้งสอง | shared | — |

- **Dead data / ของน่าสงสัย (confirm ก่อนทิ้ง):** `service_qrpayment_temp`, `transaction_xpay_fail`, `summary_deposit`, `qr_payment_confirm`, `confirm_statement` — ถูกอ้างน้อยมาก (1 ครั้ง) ⚠️ TODO: verify ว่ายังใช้จริงไหม
- **หมายเหตุ field กับดัก:** `bank_summary` filter ในโค้ด v2 ใช้ `payment_code+service` แต่ v1 ใช้ `system_code+service` (`repository/main.go:170` vs `paymentV2.go:129`) — สอง generation ของ key คนละแบบ

## 7. Variation Points (สิ่งที่ต้อง abstract ตอน rewrite)

จุดที่พฤติกรรม vary ตาม **provider** คือสาเหตุหลักที่โค้ดบวม (deposit.go 5400 บรรทัด, withdraw.go 6500 บรรทัด, callback.go 16000 บรรทัด เต็มไปด้วย `switch serviceConfig.PaymentName`). นี่คือ spec ของ abstraction ใหม่: ทุก provider ควรเป็น Strategy/plugin ที่ประกาศ auth + request mapping + callback parsing + order-no format.

| Variation | ค่าที่พบจริง | จำนวน case | นัยต่อ design |
|---|---|---|---|
| Auth/signature method | HMAC-SHA256/512, MD5, AES, JWT, RSA, Bearer, Basic, SHA | ~60 provider (matrix ด้านล่าง) | ต้องเป็น `Signer` interface ต่อ provider |
| Order-no format | plain, SHA256 ตัด 10/15/16/20 หลัก, UUID, alphanum 10 | E2P/P2CPAY=10, anypay=15, cutpayz=16, papayapay/cloudpay/visapay=20, onedaypay=UUID, trustpay=alphanum (`deposit.go:224-235`, `withdraw.go:133-142`) | abstract เป็น `OrderNoStrategy` |
| Deposit dispatch | `SwitchPaymentDeposit` switch ตาม payment_name | ~60 | plugin registry |
| Withdraw dispatch | `WithdrawAndPayOutV2` switch ตาม payment_name | ~40 (`que_payment/controllers/withdraw.go:439+`) | plugin registry |
| Callback response shape | COREPAY `{code,message}`, HENGPAY `{responseCode,responseMesg}`, default `{code,msg,data,error}` | หลายแบบ (`helper/resp.go:40`) | `ResponseFormatter` ต่อ provider |
| Callback path style | per-payment_code, one-link (deposit+withdraw รวม path), peer2pay qrStatus, xpay private | 4+ (`routes/main.go:55-68`, `routes/main.go:186`) | route abstraction |
| Queue routing | `QUE_PAYMENT_<PAYMENT_NAME>` หรือ `QUE_MERCHANT_<MERCHANT_CODE>` ถ้ามี merchant code | 2 (`service/rabbit/que-payment.go:32`) | คง logic นี้ |
| Balance source | `use_direct_balance` (ดึงจาก provider) vs bank_summary | bool flag (`withdraw.go:194`) | config flag |
| Special-case providers | PEER2PAY (bypass config), MAANPAY/XPAY (amount_actual ต่าง), HENGPAY (ห้าม user ถอน), LUCKYPAY/SWIFTPAY (ไม่รับทศนิยม) | กระจาย (`withdraw.go:104,111,225`, `deposit.go:156,388`) | edge cases ต้อง doc ครบก่อน rewrite |

**Provider auth matrix (spot-check verify แล้ว: azpay=HMAC-only ถูกต้อง, xuperma=HMAC-SHA512, speedpay=MD5 — ตรงกับ output):**

จาก grep crypto primitives ใน `3rd-payment/controller/<provider>/` (60+ โฟลเดอร์): HMAC/SHA = anypayth, askmepay, askpay, azpay, blackdog, compay(+RSA), cubixpay, hyronpay, jaijaipay, maanpay, mtmpay, okeyspay, p2cpay, peer2pay, trustpay, wealthwave, worldpay, wowpay, zappay, xuperma(HMAC-SHA512); MD5 = apay24, autopeerpay, beepay, capitalpay, cloudpay, corepay(+AES), danarapay, first2pay, k2pay, kisspay, luckythai, mtpay, mypays24, onedaypay, sonicpay, speedpay, sudahpay, sugarpay, visapay, umpay(+SHA); AES = dpay, xpay, jibpayx(+SHA), onepay(+JWT), bitpayz(+MD5+JWT); JWT/Bearer = anypay, bigpay, minerapay, thunderpay, zeenypay, e2p, swiftpay; Basic = hengpay, paykrub; ไม่พบ crypto ชัด (อาจ no-sign หรือ key ใน header) = bibpay, cutpayz, flazzpay, gbetpay, gm2pay, hashpays, hubpay, p2wpay, papayapay.

> ⚠️ Variation นี้มี 60+ instance — **ของจริงต้องแตกเป็นไฟล์ matrix แยก** (1 แถว/provider × dimension: auth, callback shape, order-no format, balance source, withdraw支持, special cases) + evidence file:line ต่อแถว. ในรอบนี้ทำได้แค่สรุป auth ที่ verify spot-check. ส่วน "ไม่พบ crypto" ต้องเปิดอ่านแต่ละตัวยืนยัน (อาจใช้ plain key).

## 8. Cross-Cutting Patterns (เก็บ pattern ไม่ใช่โค้ด)

Pattern ที่พิสูจน์แล้วว่า work และ rewrite ควร reimplement แนวคิดเดิม:

| Pattern | อยู่ที่ | ทำไมถึงดี |
|---|---|---|
| Trace propagation ผ่าน AMQP headers | inject ฝั่ง publish (`3rd-payment/service/rabbit/que-payment.go:143`), extract ฝั่ง consume (`que_payment/rabbitmqpub/amqp.go:133`) ด้วย W3C traceparent ใน `amqpcarrier` | trace ต่อเนื่องข้าม service ผ่าน queue |
| Trace propagation ผ่าน Redis (callback) | withdraw เก็บ traceparent ลง redis key `trace:cb:<orderNo>:<paymentCode>` 30 นาที (`que_payment/controllers/withdraw.go:376`) | callback ที่มาทีหลังต่อ trace เดิมได้ |
| Synchronous RPC over RabbitMQ | publish + exclusive reply queue + correlation ID + select timeout (`3rd-payment/service/rabbit/que-payment.go:78-209`) | 3rd-payment ได้ผลลัพธ์ทันที โดย delegate งานหนักไป que_payment |
| Error classifier + severity → Telegram/Mongo | `observability/classifier/classifier.go:50` แยก internal/upstream + severity 1-4, dedup 5 นาที, oncall สำหรับ critical | alert มีบริบท + ไม่ spam |
| Distributed lock (Redis) ก่อน mutate | thorlock `ObtainLock` (deposit), `is_active` flag (bank_summary), SetNX (cron) | กัน double-process เงิน |
| Ledger lock + recover cron | lock is_active ตอนถอน + cron ปลดล็อกที่ค้าง >15 นาที (`recover_bank_summary.go`) | กัน ledger ค้าง lock ถาวรถ้า process ตาย |
| No-op observability เมื่อ env ว่าง | telegram/mongo/otel ทุกตัว return no-op ถ้า config ไม่ครบ (`observability/mongodb/writer.go:47`, `init.go:13`) | service boot ได้แม้ไม่มี observability stack |
| Graceful shutdown | que_payment `ShutdownManager` รวม goroutine + drain (`que_payment/shutdown/manager.go`, `app.go:34`); 3rd-payment srv.Shutdown 60s (`app.go:173`) | drain งานก่อนตาย |
| Resty client + retry สำหรับ callback office | `newCallbackClient` retry 3 ครั้ง (`que_payment/services/callback/main.go:24`) | callback ทนต่อ network ชั่วคราว |

## 9. Non-Negotiables ✅

ของห้ามทิ้งโดยไม่มี ADR:
- ✅ **External HTTP contract** ของ deposit/withdraw/callback (section 3) — เว็บลูกค้า + 60 provider integrate อยู่ เปลี่ยน path/field = breaking
- ✅ **Money ledger invariant**: `$inc current_balance` ต้องคู่ audit `logs_inc_payment` พร้อม amount_before/after
- ✅ **Idempotency keys**: unique sparse index บน `hash` (qr_payment, withdraw_statement, adjust_statement) + deposit 5-min reuse window + status guards
- ✅ **Ledger lock discipline**: lock `is_active` ก่อนตัดยอด + refund-on-failure + recover cron
- ✅ **AMQP message contract** ระหว่างสอง repo (queue naming + ReqQue/ResQue struct)
- ✅ **Provider auth correctness ต่อราย** — sign ผิด = provider ปฏิเสธ = เงินไม่เข้า/ออก (60+ variation, section 7)
- ✅ **MongoDB shared schema** ของ collection money-path (bank_summary, qr_payment, withdraw_statement, que_payment)
- ✅ **IP whitelist บน callback** (`middleware/ipwhitelist.go`) — เปิดด้วย env `CHECK_IPWHITELIST=TRUE`
- ✅ **Trace propagation** ผ่าน AMQP + Redis (observability เป็น operational requirement)

## 10. Kill List ❌

ของที่ดูตั้งใจทิ้ง — **ทุกข้อต้อง confirm กับ owner ก่อน**:
- ❌ **v1 code paths** (DepositAgent, WithdrawAndPayOut, QueTypeSwitch, StartAMQP, UpdateSummaryByService แบบ system_code) — ดูเหมือนถูกแทนด้วย ...V2 หมดแล้ว ⚠️ confirm ว่าไม่มี caller live ก่อนทิ้ง (`que_payment/rabbitmqpub/amqp.go:225,287`)
- ❌ **AES bank-transfer gateway encryption** — ถูก comment ทิ้งทั้ง block (`routes/bank-transfer-gateway.go:17-23`, `routes/main.go:189-210`) ⚠️ confirm ว่าไม่ต้องการ encryption แล้วจริง
- ❌ **`StartCronjob` ใน 3rd-payment** — ถูก comment ใน main (`3rd-payment/_cmd/main.go:30`) ⚠️ ย้ายไป que_payment แล้ว?
- ❌ **Deposit→queue ผ่าน rabbit** — block ใหญ่ถูก comment ใน `Deposit` (`deposit.go:412-433`); deposit callback ใช้ rabbit แทน ⚠️ confirm flow ปัจจุบัน
- ❌ **CallBackWithdraw / CallbackDeposit แบบเก่า** (json marshal เอง) — ถูกแทนด้วย `CallbackAllOffice` generic, ของเก่ายังอยู่ใน `services/callback/main.go:84,123` ⚠️ มี caller ไหม
- ❌ **Dead collections** (section 6): service_qrpayment_temp, transaction_xpay_fail, summary_deposit ฯลฯ ⚠️ confirm ก่อนทิ้ง
- ❌ **legacy routes ที่ comment ไว้** ใน `routes/main.go:18-24` (speedpay direct routes)

## 11. Landmines & Mysteries ⚠️

ของที่เจอแล้วอธิบายไม่ได้ หรือเป็น bug แฝง — อันตรายสุดถ้าไม่จด:

| สิ่งที่เจอ | Evidence | สมมุติฐาน | ต้องทำอะไร |
|---|---|---|---|
| **`HashWithdraw`/`HashPayOut` คืน `hashPm` ที่คำนวณจาก `hashSha1` ตัวผิด** (ไม่ใช่ `hashSha1Pm`) | `service/hashlayout/main.go:50,71` (`hashPm := hashSha1.Sum(nil)` ทั้งที่ควรเป็น `hashSha1Pm`) | copy-paste bug — hashPm = hash (ไม่รวม paymentCode) | ถาม owner: รายการถอนพึ่ง field ไหน? ถ้าพึ่ง hashPm จริงคือ collision risk |
| **3rd-payment `UpdateSummaryByServiceV2` $inc ledger โดยไม่เขียน logs_inc_payment** | `repository/v2.go:1794` (ต่างจาก que_payment เวอร์ชันที่เขียน log) | adjust-balance flow ข้าม audit | ถาม owner: ยอดที่ปรับผ่าน 3rd-payment ต้อง audit ไหม |
| **AMQP publish ใส่ `merchant_code` แต่ consumer `ReqQue` ไม่มี field นี้** | publish `service/rabbit/que-payment.go:121`; consume `que_payment/rabbitmqpub/amqp.go:33` (ReqQue ไม่มี MerchantCode) | field ถูกทิ้งตอน unmarshal | ถ้า merchant routing สำคัญ = bug; verify ใช้จริงไหม |
| **Queue name 2 แบบไม่ sync**: publish เป็น `QUE_MERCHANT_<code>` ถ้ามี MERCHANT_CODE; consumer declare เฉพาะ `QUE_PAYMENT_<PAYMENT_NAME>` | publish `que-payment.go:32-38`; consume `amqp.go:62` | merchant-mode consumer อาจอยู่คนละ deployment | ถาม owner: มี consumer ฟัง QUE_MERCHANT_* ไหม ไม่งั้น message หาย |
| **Status เป็น magic number ทั้งระบบ ไม่มี enum/const** | 0,1,2,3,4,5,12,13,15,16,99 กระจายทั่ว callback.go/withdraw.go | ความรู้อยู่ในหัวคน | ต้องทำ status dictionary จาก owner ก่อน rewrite (section 12) |
| **status 5 (que) ถูกยอมรับใน guard แต่หา setter ไม่ชัด** | guard `objQue.Status != 2 && objQue.Status != 5` (`que_payment/controllers/withdraw.go:100,359`); withdraw_statement status 5 set ที่ `controller/withdraw.go:402` แต่ que.status 5? | สับสนระหว่าง withdraw status กับ que status | verify ว่า 5 มาจากไหนใน que_payment |
| **`helper.Resp` code semantics กลับด้าน**: code 0 บางที่ = error, บางที่ = success | success `deposit.go:171` ใช้ code 1; error `deposit.go:113` ใช้ code 0; แต่ `Resp` ส่ง HTTP 200 ทั้ง 0 และ 1 | convention ภายในไม่ชัด | ต้อง doc ว่า code 0/1 หมายถึงอะไรต่อ endpoint |
| **`MODE != production` hardcode RabbitMQ URL เป็น localhost** | `service/rabbit/que-payment.go:48` (`amqp://guest:guest@127.0.0.1:5672`) | dev shortcut | ลบตอน rewrite, ใช้ config |
| **credentials โผล่ใน docker-compose** (Mongo connection string + password) | `3rd-payment/docker-compose.yml`, `que_payment/docker-compose.yml` | ไฟล์ตัวอย่าง/dev | ห้ามนำเข้า repo ใหม่ |
| **`HashPayOut` มี special-case `PMSW` prefix** เปลี่ยน time format | `service/hashlayout/main.go:62` | swiftpay ใช้ format ต่าง | doc ไว้ |
| **`que_payment/cronjob/_retry.go` (ขึ้นต้น underscore = ไม่ compile)** เป็น ServRetryOrder เวอร์ชันไม่มี ctx | `cronjob/_retry.go:19` (signature ต่างจาก `retry-order.go:19`) | dead code (Go ignore ไฟล์ขึ้นต้น `_`) | ทิ้งได้ confirm |
| **withdraw.go 16k บรรทัดไฟล์เดียว, callback.go 16k, deposit.go 5.4k** | `wc -l controller/*.go` | ไม่มี modularization | rewrite ต้องแตกตาม provider |
| **two GetConfigPayment key generations**: v1 ใช้ system_code, v2 ใช้ payment_code | `repository/main.go:159` vs `paymentV2.go` | migration กลางคัน | verify v1 ยัง live |

## 12. Open Questions → Owner

คำถามที่โค้ดตอบไม่ได้ — **rewrite ห้ามเริ่มจนกว่าจะมีคำตอบครบ**:

1. [ ] **Status dictionary**: ความหมายที่เป็นทางการของ withdraw status 1/2/3/4/5/12/13/15/16, qr_payment status 0/1/99, que status 0/1/2 คืออะไร? มี status อื่นที่ใช้ใน production แต่ไม่อยู่ในโค้ดนี้ไหม?
2. [ ] **Source of truth ของเงิน**: `bank_summary.current_balance` เป็น authoritative ledger หรือ provider เป็น? (มี `use_direct_balance` ที่ดึงจาก provider) — กระทบ reconciliation strategy
3. [ ] **v1 paths ยัง live ไหม?** DepositAgent/WithdrawAndPayOut/StartAMQP(v1)/QueTypeSwitch — มี traffic ไหม หรือทิ้งได้
4. [ ] **Merchant-code queue routing**: มี consumer ฟัง `QUE_MERCHANT_<code>` จริงไหม? ถ้าไม่ message จะหาย (landmine)
5. [ ] **HashWithdraw/HashPayOut bug**: hashPm คำนวณผิด — เป็น bug หรือพึ่งพฤติกรรมนี้อยู่?
6. [ ] **ledger audit ฝั่ง 3rd-payment**: adjust-balance ที่ $inc โดยไม่ log — ตั้งใจหรือ bug?
7. [ ] **NFR**: throughput/latency/uptime target? deposit/withdraw ต่อวินาทีสูงสุดเท่าไร? RPC timeout 120s เหมาะไหม?
8. [ ] **Generic proxy `/call-pm/*`**: ใครเรียก? ปลอดภัยไหม (forward header+body ดิบไป provider host)?
9. [ ] **Bank-transfer gateway + AES**: ฟีเจอร์นี้ยัง active ไหม? AES encryption ที่ comment ทิ้งต้องกลับมาไหม?
10. [ ] **provider ที่ "ไม่พบ crypto"** (bibpay, cutpayz, flazzpay ฯลฯ): auth จริงคืออะไร — plain key, IP-only, หรือยังไม่ implement?
11. [ ] **callback retry / dedup**: ถ้า provider ยิง callback ซ้ำหรือ out-of-order ระบบรับมือยังไง (เห็น guard is_success แต่ไม่เห็น dedup table)?
12. [ ] **merchant/tenant model**: `merchant_code`, `system_code`, `service` ต่างกันยังไงเชิงธุรกิจ? เป็น multi-tenant แบบไหน?

## 13. Verification Plan (พิสูจน์ว่า rewrite ถูกต้อง)

- **Contract tests:** บันทึก request/response จริงของทุก endpoint money-path (section 3) เป็น golden files แล้วเทียบ byte-level; ต่อ provider ต้องมี fixture ของ callback payload จริง (60+ ราย) — สร้างจาก `logs_callback_payment` ใน production
- **Provider signing parity:** สำหรับแต่ละ provider, สร้าง test ที่ feed input เดียวกันเข้าทั้งระบบเก่า/ใหม่แล้วเทียบ signature/auth header ที่ generate (verify ว่า HMAC/MD5/AES/JWT ตรง byte ต่อ byte)
- **Parity / shadow run:** รัน service ใหม่ขนาน production รับ traffic จริง (mirror) โดยไม่ commit ledger; เทียบ status transition + ledger delta + callback payload กับระบบเก่า
- **Data reconciliation:** เขียน checker อัตโนมัติยืนยัน invariant — ทุก `$inc bank_summary` มี `logs_inc_payment` คู่ + amount_before/after สอดคล้อง; sum ของ logs = current_balance
- **Idempotency tests:** ยิง deposit/withdraw ซ้ำ key เดิม → ต้องไม่ double credit/debit (ทดสอบ unique hash index + 5-min reuse + status guard)
- **Migration gate per provider:** cutover ทีละ provider (ไม่ใช่ big-bang). เงื่อนไขก่อน cutover แต่ละราย: (1) contract test ผ่าน, (2) signing parity ผ่าน, (3) shadow run N วันไม่มี diff, (4) rollback plan พร้อม. ⚠️ TODO: owner กำหนด baseline diff ที่ยอมรับได้ + ลำดับ provider
- **Status semantics gate:** ห้าม rewrite state machine จนกว่า status dictionary (Q1) ได้รับการยืนยัน

## 14. Deep-Dive Docs (ถ้าโปรเจคใหญ่)

| หัวข้อ | ไฟล์ |
|---|---|
| Provider variation matrix (60+ ราย × auth/callback/orderno/withdraw) | ⚠️ ยังไม่สร้าง — รอบนี้สรุปย่อใน section 7; ของจริงควรแตกเป็น `02-providers-matrix.md` |
| External contract per-endpoint (80+ routes) | ⚠️ ยังไม่สร้าง — ควรแตกเป็น `03-contracts/` ต่อกลุ่ม (deposit/withdraw/callback/dashboard/bot) |
| Status dictionary (รอ owner) | ⚠️ รอ section 12 Q1 |
| Provider docs ที่มากับ repo (ตรวจกับโค้ดก่อนเชื่อ) | `3rd-payment/doc/` (xuperma, cubixpay, danarapay, deposit-flow) — **อย่าเชื่อ doc, verify กับโค้ด** |
