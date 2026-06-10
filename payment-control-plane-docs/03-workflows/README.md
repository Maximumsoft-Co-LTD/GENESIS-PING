---
last_reviewed: 2026-06-10
---

# 03-workflows

Step-by-step flow ที่ข้าม service

ใช้ template: `../templates/workflow.md`

## ที่เขียนแล้ว

- ✅ `payin-end-to-end.md` — merchant → QR → callback → settle → webhook (3 legs)
- ✅ `payout-end-to-end.md` — create → confirm → queue → provider (two-phase + compensation)
- ✅ `queue-task-lifecycle.md` — hybrid queue + dispatcher 8 types + ⚠️ poll path re-publish เข้า MQ

## Workflow ที่เหลือ (เขียนเมื่อ rewrite แตะ)

1. `callback-inbound-flow.md` — provider hit → IPwhitelist → lock → update state (บางส่วนอยู่ใน payin Leg B แล้ว)
2. `callback-outbound-retry-flow.md` — ครอบคลุมแล้วใน `04-integrations/office-webhook.md` ส่วน Retry
3. `slip-verify-flow.md`
4. `reconciliation-flow.md` — `check/order` → fix state
5. `manual-confirm-flow.md` — office confirm deposit ตกค้าง (status 99)
