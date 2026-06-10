---
last_reviewed: 2026-06-10
---

# 03-workflows

Step-by-step flow ที่ข้าม service

ใช้ template: `../templates/workflow.md`

## Workflow ที่ควรเขียนก่อน

1. `payin-end-to-end.md` — merchant → 3rd-payment → queue → que_payment → provider → callback → outbound webhook
2. `payout-end-to-end.md`
3. `callback-inbound-flow.md` — provider hit → IPwhitelist → lock → update state → notify merchant
4. `callback-outbound-retry-flow.md` — first send → fail → cron retry
5. `slip-verify-flow.md`
6. `reconciliation-flow.md` — `check/order` → fix state
7. `manual-confirm-flow.md` — office staff confirm deposit ตกค้าง (status 99)
