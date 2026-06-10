---
name: config-plane
type: feature
status: active
last_reviewed: 2026-06-10
implementations:
  - 3rd-payment: model/service_payment.go:10-80, repository/v2.go:111-121
  - que_payment: models/service_payment.go:5-59, repository/paymentV2.go:18-24
related_workflows:
  - 03-workflows/payin-end-to-end.md
---

# Feature: `config-plane`

> **หนึ่งบรรทัด:** provider config ทั้งหมดอยู่ใน MongoDB collection `service_qrpayment` — struct `ServicePayment` ~50 field คือ "ทุกอย่างที่ระบบรู้เกี่ยวกับ provider หนึ่งตัว"

## 1. Business Definition

เพิ่ม/แก้ provider = แก้ row ใน DB (ผ่าน create/update-config-payment API) ไม่ต้อง deploy — rewrite thesis ข้อ 6 ต้องการขยายแนวคิดนี้ให้ adapter โหลดจาก config ทั้งหมด

## 2. Cross-Provider Pattern Matrix

Schema เดียวใช้ทุก provider แต่ field จำนวนมาก**เป็นของ provider เฉพาะตัว** (comment ในโค้ดบอกเอง):

| กลุ่ม field | ตัวอย่าง | หมายเหตุ |
|---|---|---|
| Identity | `payment_code` (PK lookup), `payment_name`, `payment_type` (WITHDRAW_ONLY / DEPOSIT_ONLY / DEPOSIT_AND_WITHDRAW), `custom_name`, `service`, `merchant_code`, `system_code` | |
| Credentials ทั่วไป | `merchant_no`, `access_key`, `secret_key`, `key_code`, `client_id` | |
| Credentials เฉพาะเจ้า | `merchant_id`/`merchant_token`/`merchant_secret`/`identity_token` (SWIFTPAY), `partner_key`/`iv_code`/`pm_token` (corepay) | ⚠️ schema บวมเพราะ per-provider field |
| URLs | `host_smart_pay`, `api_qrpay`, `path_callback_deposit`, `path_callback_withdraw`, `rabbitmq_url`, `office_api`, `payment_detail_url`, `uri_forward_payin/payout` | `rabbitmq_url` ต่อ config! (ดู tech-stack) |
| Flags | `status` (1=active — เงื่อนไข lookup), `use_direct_balance`, `is_peer_to_peer`, `is_approve_bank`, `check_bank_number`, `split_active`, `member_register` (corepay) | |
| Fees/Limits | `fee_deposit`, `fee_withdraw`, `fee_qrpay`, `add_money`, `min/max_deposit`, `min/max_withdraw` | |
| Nested | `service_deposit` (12 fields รวม username/password!), `service_withdraw` (รวม acc no/credentials), `deposit_type` (การ render QR), `option` (9 toggle), `telegram_bot_setting`, `service_whitelist` | |
| Security | `whitelist_ip` — **type `interface{}`** ใน struct หลัก แต่ `[]string` ใน create body ⚠️ | |

## 3. Common Behavior (DNA)

1. **Lookup ตอน runtime**: `GetConfigPaymentV2(paymentCode)` — filter `{payment_code, status: 1, system_code}` (`3rd-payment/repository/v2.go:111-121`) — โหลดจาก Mongo **ทุก request ไม่มี cache**
2. Authorization ของ merchant = `service_whitelist` ใน config (`middleware.CheckServiceWhitelist`)
3. Create flow มี staging: เขียน `service_qrpayment_temp` ก่อน (validation) แล้วค่อยเข้า `service_qrpayment` (`repository/v2.go:74-88`)
4. Create ป้องกันด้วย header `X-Auth-Token` เทียบ env `ALLOWED_CONFIG_KEYS` (`controller/main.go:3070-3077`)

## 4. Variation Points (Strategy)

- **Indonesia variant** (`isIndo=true` จาก route `create-indo-config-payment` — `routes/main.go:39`): (a) payment_code ใช้ค่าที่ส่งมาได้, (b) duplicate check เช็คแค่ payment_code, (c) **ข้าม X-Auth-Token auth ทั้งหมด** ⚠️ (`controller/main.go:3070,3118,3192`) — สรุป: **เป็น flag ไม่ใช่ product แยก** (ตอบ open question ข้อ 1 ของ RECAP ได้บางส่วน — เหลือ confirm กับ owner)
- **Drift ระหว่าง repo**: `ServicePayment` ฝั่ง que เป็น subset (~ขาด 15+ field: TelegramBotSetting, SettingForward, audit timestamps, Option บางตัว) แต่มี field เพิ่ม 2 ตัว (`BankSupport []string`, `LogoUrl`) — และ query ฝั่ง que ไม่ filter `system_code` (`que_payment/repository/paymentV2.go:18-24`) ⚠️

## 5. Invariants

- `payment_code` unique (Indo path เช็คตรงๆ, ปกติเช็ค composite)
- `status=1` เท่านั้นที่ถูกใช้งาน — ปิด provider = set status
- que_payment อ่านอย่างเดียว (ยกเว้น update เล็กน้อย 3 จุด — ดู collection-ownership)

## 6. Data Touched

| Store | Object | Op |
|---|---|---|
| MongoDB | `service_qrpayment` | 3rd: R/W (CRUD ผ่าน API) · que: R (+W 3 จุด) |
| MongoDB | `service_qrpayment_temp` | 3rd: W (staging) |
| Redis | `payment_bank_support` (cache รายชื่อธนาคาร), token bigpay | clear ผ่าน endpoint `/clear-redis` |

## 7. Failure Modes

| Mode | What happens | Mitigation |
|---|---|---|
| Config ไม่เจอ / status!=1 | ทุก op ตอบ "channel unavailable" | — |
| แก้ config ระหว่างมี order ค้าง | พฤติกรรมเปลี่ยนกลางคัน (โหลดสดทุก request) | rewrite: config version/snapshot per order |
| `whitelist_ip` เป็น interface{} | type ผิดได้เงียบๆ | rewrite: typed []string |

## 8. Rewrite Notes

- 🟢 Keep: DB-driven config, status flag, staging table concept, X-Auth-Token gate
- 🔴 Drop: per-provider field ระเบิดใน struct เดียว → ย้ายเป็น `credentials JSON` per adapter type; Indo bypass auth
- 🟡 Reshape: lookup ทุก request → cache + invalidation; แยก merchant-facing config (whitelist, office_api) ออกจาก provider credentials
- Suggested: `config_payment` (identity+flags+limits) + `provider_credentials` (typed per adapter) + `merchant_binding` (service↔payment_code whitelist)

## 9. Open Questions

- [ ] Indo bypass auth — ตั้งใจไหม? ⚠️ TODO: verify with @owner
- [ ] `service_qrpayment_temp` ใครเป็นคน approve จาก temp → จริง? (ไม่พบ promote code ชัดๆ)
- [ ] field ที่ que มีเพิ่ม (`BankSupport`, `LogoUrl`) ใครเขียน?
