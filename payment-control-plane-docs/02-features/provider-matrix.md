---
name: provider-matrix
type: feature
status: active
last_reviewed: 2026-06-10
implementations:
  - 3rd-payment: controller/<name>/ (~66 folders)
  - que_payment: services/<name>/ (62 folders)
related_workflows:
  - 03-workflows/payin-end-to-end.md
  - 03-workflows/callback-inbound-flow.md
---

# Feature: `provider-matrix`

> **หนึ่งบรรทัด:** ตารางสรุปว่า provider ทั้ง 65 ตัวเหมือน/ต่างกันตรงไหน — ข้อมูลชี้ขาด design ของ Provider interface ตอน rewrite

## 1. Business Definition

ทุก provider คือ "ตัวกลางรับ-จ่ายเงิน" ที่ระบบต้องคุยด้วยผ่าน HTTP API ของเขา
ความต่างระหว่าง provider (auth, signature, callback, time format) คือสาเหตุที่โค้ดปัจจุบันมี 60+ โฟลเดอร์ซ้ำ 2 repo
Matrix นี้ระบุ variation ทั้งหมด เพื่อให้ rewrite ออกแบบ abstraction เดียวที่ครอบทุกตัวได้

**วิธีอ่าน:**
- **Ops** — payin (ฝาก), payout (ถอน), cb-in (รับ callback), query (เช็คสถานะ), balance, stmt (statement/report)
- **CB verify** — provider ตรวจ callback ขาเข้ายังไง: `sig` = verify signature, `token` = เทียบ token, `none` = พึ่ง IP whitelist อย่างเดียว, `?` = หาไม่เจอในโค้ด
- `?` = ยังหาคำตอบจากโค้ดไม่ได้ — ⚠️ TODO: verify with @owner

## 2. Cross-Provider Pattern Matrix

### Batch A-B (anypay → bitpayz)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| anypay | both | payin, payout, balance | Bearer/login-token | n/a (token) | ? | split | n/a | code-field | token-based API |
| anypayth | both | payin, payout, balance | HMAC-SHA256 | timestamp\|refNo\|amount\|status\|currency\|type\|date | sig | split | unix-ms | code-field | 5-min timestamp window |
| apay24 | both | payin, payout, balance | basic (base64) + MD5-concat | order_no+secret | ? | split | n/a | HTTP-status | MD5 deposit hash validation |
| askmepay | both | payin, payout, balance, cb-in | HMAC-SHA256 | JSON body | sig | one-link | unix | code-field | cashier redirect flow |
| askpay | both | payin, payout, query, balance | HMAC-SHA256 | timestamp\nmethod\npath\nbodyHash | sig | one-link | unix-ms | HTTP-status | canonical request string |
| autopeerpay | both | payin, payout, balance | basic + MD5-concat | order_no+secret | ? | split | RFC3339 | HTTP-status | slip upload flow; โค้ดเหมือน apay24 |
| azpay | both | payin, payout, balance | HMAC-SHA256 | raw body bytes | sig | one-link | n/a | code-field (status_code=200) | sign ตรงด้วย secret |
| beepay | both | payin, payout, query | MD5-concat | param string + key | sig | split | "20060102150405" local | code-field | MD5 parameter chain |
| bibpay | both | payin, payout, cb-in | JWT/custom | UUID gen | token | one-link | UTC datetime | code-field (status) | custom JWT flow |
| bigpay | both | payin, payout, balance, query, cb-in | Bearer/login-token | n/a (token) | ? | split | n/a | code-field (success) | Redis token caching |
| bitpayz | both | payin, balance | JWT (external endpoint) | timestamp + payload | ? | one-link (smartpay) | unix + unix-ms | code-field | AES-CBC decrypt; JWT จาก external service |

### Batch B-F (blackdog → first2pay)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| blackdog | both | payin, payout, query | HMAC-SHA256 | payload + secret | sig | split | unix | HTTP-status | webhook timestamp freshness 5m |
| capitalpay | both | payin, payout, query, balance | MD5-concat | sorted params + secret | ? | split | ? | code-field (message) | — |
| cloudpay | both | payin, query, balance | RSA (external NodeEncrypt) | built map fields | sig (RSA) | split | "20060102150405" | code-field (status!=SUCCESS) | sign ผ่าน external encryption service |
| compay | both | payin, payout, query, balance | HMAC-SHA256 | apiKey:key=val&... | sig | split | RFC3339 (.000Z) | HTTP-status | header x-signature + x-api-key |
| corepay | both | payin, payout, query, balance | MD5 double-hash | token\*\|\*sorted+@!@timestamp | ? | split | unix | code-field | AES encrypt body; timestamp ใน signature |
| cubixpay | both | payin, payout, query | HMAC-SHA256 | sorted params + timestamp | sig (SHA256 plain) | split | unix | code-field (success bool) | payout ใช้ X-Signature header |
| cutpayz | both | payin, query, balance | API-key-header (x-api-key) | ? | ? | split | ? | code-field | x-api-key derive จาก merchant+secret |
| danarapay | 3rd-only | payin | API-key-header | ? | ? | ? | ? | code-field (status.code) | x-username + x-api-key; เพิ่งเข้า (PR ล่าสุด) ยังไม่มีฝั่ง que |
| dpay | both | payin, payout, query, balance | API-key-header (CLIENT-ID/SECRET) | ? | ? | split | ? | code-field (StatusCode!=200) | — |
| e2p | both | payin, payout, query, balance | SHA256-concat | merchant_id+order_no+token | ? | split | ? | code-field (status!=succeeded) | X9K variant (path adjust) |
| first2pay | both | payin, payout, query, balance | basic + MD5-sign | sorted fields + secret | sig | split | ? | code-field (code!=200) | เจ้าของ endpoint `virify/slip` |

### Batch F-J (flazzpay → jibpayx)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| flazzpay | 3rd-only | payin, query | SHA512-concat | compact JSON + secret | none | ? | ? | code-field | compact JSON signing |
| gbetpay | both | payin, payout, balance | API-key-header (x-auth-token) | ไม่มี signature | ? | ? | ? | ? | secret ใส่ header ตรงๆ |
| gm2pay | both | payin, payout, query, balance | API-key-header | X-API-Key + X-API-Secret headers | ? | ? | ? | HTTP-status | headers-based auth |
| hashpays | 3rd-only | payin | HMAC-SHA256 | unix-ms + body | ? | ? | unix-ms | ? | crypto deposit only |
| hengpay | both | payin, payout, balance, query, stmt | Bearer/login-token | access_token header | ? | split + URL พิเศษ | RFC3339 | code-field | ต้อง login ก่อน; มี withdraw-callback URL แยก + admin routes (update-callback ฯลฯ) |
| hubpay | both | payin, payout, balance | API-key-header (X-API-Key) | ไม่มี signature | ? | ? | ? | code-field (success) | no signing |
| hyronpay | both | payin, payout, balance | HMAC-SHA256 | timestamp.body | sig (X-Webhook-Signature) | split | unix | HTTP-status | — |
| impay | both | payin, payout, query | SHA256-concat | client_id+...+secret | ? | split | ? | code-field | concatenation signing |
| jaijaipay | both | payin, payout, balance, query | HMAC-SHA256 | timestamp + body | ? | split | unix | code-field | มี route `request-resend-callback` เฉพาะ |
| jibpayx | both | payin, payout, balance, query | API-key-header + AES | x-api-key + AES encrypted data | ? | split | unix | code-field (success) | AES + PBKDF2 key derivation |

### Batch K-O (k2pay → onepay)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| k2pay | both | payin, payout, query, balance | MD5-concat | appid+order+...+secret | sig | split | n/a | code-field (Error/Code!=0) | MD5 ทุก op |
| kisspay | both | payin, query, balance | MD5-concat | timestamp+apiKey | sig | split | unix decimal (UnixNano/1e9) | code-field | timestamp เป็น float |
| luckythai | both | payin, payout, query, balance | MD5-concat | clientCode&clientNo&orderNo&payAmount&status&txid+key | sig | split | unix-ms | code-field (Code!=0) | blockchain fields (chainName, coinUnit) |
| maanpay | both | payin, payout, query, balance | HMAC-SHA256 | sorted JSON + timestamp | ⚠️ sig **ถูก comment ทิ้ง** | split | unix | code-field (Code!=0) | callback verify ปิดอยู่ในโค้ด |
| minerapay | both | payin, payout, query, balance | Bearer/login-token | email+password → token | none | split | n/a | code-field | token cache ใน Redis; callback ไม่ verify |
| mtmpay | both | payin, payout, query | HMAC-SHA256 | timestamp\|method\|path\|body | none | split | unix | HTTP-status | x-signature + x-timestamp headers |
| mtpay | both | payin, payout, cb-in, query, balance | MD5-concat | secret+orderID (callback) | token (MD5) | split | n/a | code-field | callback token = MD5 |
| mypays24 | both | payin, payout, query, balance | basic + MD5 | order_no+secret_key | none | split | n/a | code-field | base64 Basic Auth |
| okeyspay | 3rd-only | payin, payout, query, balance | HMAC-SHA256 | base64(sorted params)\|timestamp | none | split | unix-ms | code-field (Message!=success) | base64 ใน sig input |
| onedaypay | both | payin, payout, cb-in, query, balance | MD5 pattern | key:money:...:secret | none | split | unix | code-field | sig แบบ pattern colon-joined |
| onepay | both | payin, payout, query, balance | JWT | timestamp+clientID (จาก /jwt/create) | none | split | unix-ms | code-field | ขอ JWT จาก endpoint ของ provider |

### Batch P-S (p2cpay → sugarpay)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| p2cpay | both | payin, payout, cb-in, query, balance | HMAC-SHA256 | JSON body + secret | sig | split | unix-ms | code-field (success bool) | duplicate-ref fallback lookup; route `/p2cpay/channels` |
| p2wpay | both | payin, balance | API-key-header | ไม่มี signature | none | split | ? | code-field (success bool) | no signing |
| papayapay | both | payin, payout, cb-in | Bearer/token-header | transactiontoken เท่านั้น | token | split | ? | code-field (StatusCode) | endpoint encryption |
| paykrub | both | payin, payout, cb-in, query | MD5-concat | concat params + secret | token | split | unix | code-field (status) | มี SHA1 fallback ใน helper |
| peer2pay | both | payin, payout, cb-in | HMAC-SHA256 | len(body)+accessKey | sig | qrStatus | unix | code-field (code!=0000) | namespace `/peer2pay/v3` + p2p endpoints; ใช้ ENCRYPTION_ENDPOINT |
| sonicpay | both | payin, payout, cb-in | MD5-concat | concat signData + key | token | split | ? | code-field (status) | auth token fetch |
| speedpay | both | payin, payout, cb-in, query, balance | MD5-concat | sorted params + key | token | split | "20060102150405" | code-field | มีซาก route เก่า comment ทิ้งใน routes/main.go |
| sudahpay | both* | payin, cb-in, query | MD5 | id+amount+merchant_id+channel+secret+project_no | sig | split | ? | code-field (ErrorCode P00) | *ฝั่ง que อยู่ใน `service/` (เอกพจน์) ไม่ใช่ `services/` |
| sugarpay | both | payin, payout, cb-in, balance | SHA256 + basic | sorted JSON + secret | sig | split | ? | code-field (status) | Basic auth header (token:) |

### Batch S-Z (swiftpay → zpay)

| Provider | Repos | Ops | Auth | Sign input | CB verify | CB style | Time fmt | Err shape | Quirks |
|---|---|---|---|---|---|---|---|---|---|
| swiftpay | both | payin, payout, cb-in, query, balance | Bearer/login-token + sign | sorted fields + merchant_secret | sig | split | RFC3339 | code-field (code!=0) | OrderEnquiry ผ่าน GET |
| thunderpay | both | payin, payout, cb-in, query, balance | JWT (HS256) | JWT payload (merchantId + clientId) | sig (JWT) | split | unix-ms | code-field (status) | signature คือ JWT token |
| trustpay | both | payin, payout, cb-in, balance | HMAC-SHA256 | path + body + nonce (hex) | sig | split | n/a | code-field (status) | nonce = base64(HMAC-SHA256) |
| umpay | both | payin, payout, cb-in, query, balance | MD5-concat | transaction_id+username+amount+secret | sig (MD5) | split + URL พิเศษ | ? | code-field (Code!=0) | `/callback-payout/umpay/...` + `/check-uuid/umpay/...` |
| visapay | both | payin, payout, cb-in, balance | RSA (external encrypt) | sorted params query-string | sig (RSA decrypt) | split | ? | code-field (success) | encrypt/decrypt ผ่าน ENCRYPTION_ENDPOINT (Node.js) |
| wealthwave | both | payin, payout, cb-in, query, balance | HMAC-SHA256 | JSON body | sig | split | unix | code-field (Code!=0) | — |
| worldpay | both | payin, payout, cb-in, query, balance | HMAC-SHA256 | timestamp\|method\|path\|body | sig | split | unix-ms | code-field | headers x-api-key/x-signature/x-timestamp |
| wowpay | both | payin, payout, cb-in, query, balance | HMAC-SHA256 | method:path:unix-ms:base64(sorted body) | sig | split | unix-ms | ? | auth header format "Y3-HMAC-SHA256" |
| xpay | both | payin, payout, cb-in, balance | AES (CFB) | AES decrypt callback data | sig (AES decrypt) | split | ? | ? | namespace แยก `/api/xpay/private` |
| xuperma | 3rd-only | payin, payout, cb-in, balance | HMAC-SHA512 | key + json.encode(body) | sig | split | ? | ? | escape "/" เป็น "\\/" เลียนแบบ PHP json_encode |
| zappay | both | payin, payout, cb-in, query, balance | HMAC-SHA256 | client_id\|timestamp\|body\|query | sig | split | unix | ? | คล้าย worldpay แต่รวม query string |
| zeenypay | both | payin, payout, cb-in, query, balance | basic (base64 secret) | base64(secret+":") | token | split | ? | code-field (status) | basic auth |
| zpay | both | payin, payout, cb-in, query | SHA256-concat | merchant_code&merchant_key&currency&payment_id&response_url&amount | sig | split | ? | ? | signature string คั่นด้วย & |

> Evidence (file:line ของ auth แต่ละตัว) อยู่ท้ายไฟล์ — ดู section "Evidence"

## 3. Common Behavior (DNA)

สิ่งที่ (เกือบ) ทุก provider ทำเหมือนกัน:
1. **โครงไฟล์เหมือนกันหมด**: `main.go` (entry ops) + `func.go` (signing/HTTP helpers) + `model.go` (req/res structs) — ทั้ง 2 repo
2. **ทุก op รับ config จาก MongoDB** (`service_payment` / `config_payment`) — credentials, URL, path ของ callback อยู่ใน DB ไม่ hardcode (ยกเว้น quirk บางตัว)
3. **Order lookup ด้วย `order_no` + status check ก่อน mutate** — เป็น idempotency แบบ implicit ทุกตัว (ไม่มีตัวไหนใช้ idempotency key จริงจัง)
4. **เรียก provider API → แปลง response → เขียน MongoDB → ยิง outbound callback ไป merchant office** — ลำดับเดียวกันหมด
5. **ไม่มี provider ไหนจัดการ retry เอง** — พึ่ง cronjob กลาง (`ServRetryOrder`, `AutomationRetryCallbackStatus`)

## 4. Variation Points (Strategy)

จุดที่ provider ต่างกัน → ต้องเป็น strategy/config ใน abstraction ใหม่:

| Variation | ค่าที่พบ (ตัวอย่าง) | นัยต่อ design |
|---|---|---|
| **Auth/sign ขาออก** | HMAC-SHA256 (มากสุด ~40%), MD5-concat, SHA256/512-concat, API-key header เปล่า, Bearer/login-token, JWT, RSA external, AES, basic | ต้องมี `Signer` interface + standard implementations |
| **Sign input** | raw body, sorted params, canonical string (ts\|method\|path\|body), field concat | ห้าม hardcode — เป็น per-provider strategy |
| **CB verify ขาเข้า** | signature-verify, token-compare, **none (IP whitelist อย่างเดียว) — เยอะมากในกลุ่มเก่า** | rewrite ต้องบังคับ verify ทุกตัวที่ทำได้ |
| **CB style** | split deposit/withdraw (ส่วนใหญ่), one-link (askmepay, askpay, azpay, bibpay, bitpayz, ...), URL พิเศษ (hengpay, umpay, xpay, peer2pay) | one URL + dispatch by payload ตาม kill list |
| **Time format** | unix, unix-ms, RFC3339, "20060102150405" local, UTC datetime | ต้องมี per-provider time codec |
| **Error semantics** | code-field (หลากหลาย: code!=0, status!=SUCCESS, success bool), HTTP-status, string-match | ต้อง normalize เป็น error enum กลาง |
| **External crypto service** | cloudpay, visapay (RSA), bitpayz (JWT) เรียก `ENCRYPTION_ENDPOINT` | dependency แฝง — ต้องตัดสินใจว่า inline หรือคง service แยก |

## 5. Invariants

- `order_no` unique ต่อ merchant — ทุก provider ใช้เป็น key หลัก
- ทุก inbound callback ผ่าน `middleware.IPWhitelist` ก่อนเสมอ (แม้ตัวที่ verify signature เองด้วย)
- ทุก mutate ฝั่ง 3rd-payment อยู่ใต้ thorlock (ฝั่ง que ไม่มี — ดู open question)
- Response ที่ตอบ provider ขาเข้า ต้องตามสเปคของแต่ละเจ้า (บางเจ้าต้องการ "SUCCESS" string, บางเจ้า JSON)

## 6. Data Touched

| Store | Object | Op |
|---|---|---|
| MongoDB | `service_payment` / `config_payment` | R (credentials, URLs) |
| MongoDB | `statement`, order collections | R/W (ทุก op) |
| MongoDB | `bank_summary` | W (balance ops) |
| MongoDB | `que_payment` | W (3rd) / R (que poll) |
| Redis | token cache (bigpay ฯลฯ), thorlock | R/W |
| External | provider API, ENCRYPTION_ENDPOINT (บางตัว) | call |

## 7. Failure Modes

| Mode | What happens | Mitigation |
|---|---|---|
| Provider ตอบ 200 แต่ code=error | ตีความผิดถ้าเช็คแค่ HTTP status | normalize error ใน adapter |
| Callback ปลอม (ตัวที่ไม่ verify sig) | พึ่ง IP whitelist อย่างเดียว | rewrite: บังคับ signature verify |
| Timestamp window เกิน (anypayth 5min, blackdog 5m) | callback ถูก reject | NTP sync + retry |
| External encryption service ล่ม | cloudpay/visapay/bitpayz ตายทั้งสาย | ตัดสินใจ inline crypto ตอน rewrite |
| แก้โค้ด provider ฝั่งเดียว (3rd หรือ que) | drift — bug เฉพาะ async path | รวม adapter เดียว (เป้าหมายหลัก rewrite) |

## 8. Rewrite Notes

- 🟢 Keep: โครง 3 ไฟล์ต่อ provider (มันคือ proto-adapter อยู่แล้ว), order_no-based lookup
- 🔴 Drop: โค้ดซ้ำ 2 repo, URL พิเศษ per-provider, IP-whitelist-only callbacks
- 🟡 Reshape: signing → `Signer` strategy, error → enum กลาง, time → per-provider codec

Suggested abstraction:
```go
type Provider interface {
    CreatePayin(ctx context.Context, req PayinReq) (PayinRes, error)
    CreatePayout(ctx context.Context, req PayoutReq) (PayoutRes, error)
    VerifyCallback(ctx context.Context, raw []byte, h http.Header) (Event, error) // บังคับทุกตัว
    QueryOrder(ctx context.Context, ref OrderRef) (OrderStatus, error)
    Balance(ctx context.Context) (Balance, error)
}

// Variation points แยกเป็น strategy ที่ inject เข้า adapter:
type Signer interface{ Sign(req *SignContext) error }       // HMAC/MD5/RSA/JWT/...
type ErrorDecoder interface{ Decode(res *http.Response) error } // code-field/HTTP-status/...
```

## 9. Open Questions

- [ ] Provider ที่ CB verify = `?` หรือ `none` (~20 ตัว ส่วนใหญ่กลุ่ม K-O) — ของจริง verify ที่ไหนสักแห่งไหม หรือพึ่ง IP whitelist จริงๆ? ⚠️ TODO: verify with @owner
- [ ] **maanpay: callback signature verification ถูก comment ทิ้ง** (`maanpay/func.go:56-59` บริเวณ callback path) — ตั้งใจปิดชั่วคราวหรือลืม? ⚠️ TODO: verify with @owner
- [ ] `ENCRYPTION_ENDPOINT` (Node.js service — cloudpay, visapay, peer2pay, bitpayz ใช้) — ใครดูแล? จะ inline เป็น Go crypto ตอน rewrite ได้ไหม?
- [ ] Provider 3rd-only 5 ตัว (danarapay, flazzpay, hashpays, okeyspay, xuperma) — ยังใช้งานจริงไหม หรือรอ port ฝั่ง que? (danarapay เพิ่งเข้าใน PR ล่าสุดของทั้ง 2 repo แต่โฟลเดอร์ que ยังไม่มี)
- [ ] hashpays = "crypto deposit only", luckythai มี blockchain fields — เป็น product line แยกไหม?

---

## Evidence (auth claims)

<!-- file:line ของ signing code แต่ละ provider — รวบรวมจากการ audit 2026-06-10 -->
- anypay: `3rd-payment/controller/anypay/func.go:41` · anypayth: `anypayth/func.go:58-60` · apay24: `apay24/func.go:44-53` · askmepay: `askmepay/func.go:62-65` · askpay: `askpay/func.go:15-21` · autopeerpay: `autopeerpay/func.go:44-53` · azpay: `azpay/func.go:15-18` · beepay: `beepay/func.go:8-12` · bibpay: `bibpay/func.go:3` · bigpay: `bigpay/main.go:47` · bitpayz: `bitpayz/func.go:167-180`
- blackdog: `blackdog/main.go:24-44` · capitalpay: `capitalpay/func.go:18-42` · cloudpay: `cloudpay/main.go:47-63` · compay: `compay/func.go:127-135` · corepay: `corepay/func.go:21-46` · cubixpay: `cubixpay/func.go:45-77` · cutpayz: `cutpayz/func.go:60` · danarapay: `danarapay/func.go:33-35` · dpay: `dpay/main.go:47-48` · e2p: `e2p/func.go:1-30` · first2pay: `first2pay/func.go:18-59`
- flazzpay: `flazzpay/func.go:12-16` · gbetpay: `gbetpay/func.go:21,34` · gm2pay: `gm2pay/main.go:34-35` · hashpays: `hashpays/func.go:26-34` · hengpay: `hengpay/func.go:41-46` · hubpay: `hubpay/main.go:29-33` · hyronpay: `hyronpay/func.go:45-51` · impay: `impay/func.go:15-29` · jaijaipay: `jaijaipay/func.go:74-89` · jibpayx: `jibpayx/func.go:176-197`
- k2pay: `k2pay/func.go:17-20` · kisspay: `kisspay/func.go:14-30` · luckythai: `luckythai/func.go:16-21` · maanpay: `maanpay/func.go:56-59` · minerapay: `minerapay/func.go:38-78` · mtmpay: `mtmpay/func.go:69-84` · mtpay: `mtpay/main.go:275-276` · mypays24: `mypays24/func.go:45-49` · okeyspay: `okeyspay/func.go:27-53` · onedaypay: `onedaypay/func.go:11-22` · onepay: `onepay/func.go:16-36`
- p2cpay: `p2cpay/func.go:16-24` · p2wpay: `p2wpay/main.go:32-34` · papayapay: `papayapay/func.go:28-29` · paykrub: `paykrub/func.go:69-80` · peer2pay: `peer2pay/func.go:78-82` · sonicpay: `sonicpay/func.go:163-168` · speedpay: `speedpay/func.go:12-16` · sudahpay: `sudahpay/func.go:18-21` · sugarpay: `sugarpay/func.go:22-46`
- swiftpay: `swiftpay/func.go:89` · thunderpay: `thunderpay/func.go:24` · trustpay: `trustpay/sign.go:29` · umpay: `umpay/func.go:104` · visapay: `visapay/func.go:113-183` · wealthwave: `wealthwave/func.go:16` · worldpay: `worldpay/func.go:78` · wowpay: `wowpay/func.go:17` · xpay: `xpay/func.go:79` · xuperma: `xuperma/func.go:16` · zappay: `zappay/func.go:70` · zeenypay: `zeenypay/func.go:18` · zpay: `zpay/func.go:21`
