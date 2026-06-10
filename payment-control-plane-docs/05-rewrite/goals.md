---
last_reviewed: 2026-06-10
---

# Rewrite Goals

## ทำไมต้อง rewrite (business case)

1. **Duplication 59 provider × 2 repo** — แก้ feature หนึ่งต้องแก้ 2 ที่ drift สะสม (เห็นแล้วจริงใน schema: `ServicePayment` สอง repo ไม่เท่ากัน, query ต่างกัน)
2. **เพิ่ม provider ใหม่แพง** — ต้องแตะ routes, controller, services, config ใน 2 repo
3. **ความเสี่ยงการเงินที่พบจาก audit:**
   - callback ~20 provider ไม่ verify signature (พึ่ง IP whitelist)
   - outbound webhook ไป merchant ไม่มี signature เลย
   - worker ไม่มี distributed lock (thorlock ถูก comment ทิ้ง)
   - hybrid queue ไม่ standalone จริง — Rabbit ล่ม = ทุกอย่างหยุด
   - merchant API ไม่มี authentication จริง (แค่ service whitelist)
4. **Operational debt** — V1/V2 routes คู่, test endpoints ใน prod, lock กระเป๋าไร้ TTL, retry ไร้ DLQ

## เป้าหมาย (เรียง priority — ตรงกับ REWRITE-PROMPT DESIGN GOALS)

1. Kill the 59×2 duplication — adapter เดียวใช้ทั้ง HTTP และ worker
2. เพิ่ม provider ใหม่ = งาน ~1 วัน (config + adapter registration)
3. Idempotency เป็น first-class (วันนี้: hash index + final-state check — ดี แต่ implicit)
4. Outbound webhook reliable (signature + retry + DLQ ที่มองเห็น)
5. เก็บของดี: OTel/AMQP propagation, severity Telegram, batch event writer, shutdown manager
6. ลด surface: V1 routes, URL พิเศษ per-provider, debug endpoints
7. No big-bang — migrate ทีละ provider ด้วย feature flag

## Definition of Done ของ docs package (เฟสถอด DNA)

- [x] Provider matrix 65 ตัว (`02-features/provider-matrix.md`)
- [x] Merchant API + webhook contract (`04-integrations/`)
- [x] Config plane schema (`02-features/config-plane.md`)
- [x] Order state machine (`02-features/order-state-machine.md`)
- [x] Collection ownership (`02-features/collection-ownership.md`)
- [x] Workflows payin/payout/queue (`03-workflows/`)
- [ ] คำตอบ open questions จาก owner (กรอกใน REWRITE-PROMPT ก่อนเริ่ม)
- [ ] NFR: throughput, latency, deployment topology (owner)
