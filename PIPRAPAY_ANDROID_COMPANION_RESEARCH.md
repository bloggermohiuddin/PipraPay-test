# PIPRAPAY Android Companion Research

## 1. Project Overview
PipraPay in this repository is **not a Laravel codebase** (no `routes/api.php`, `routes/web.php`, Controllers/Models/Middleware directories). It is a custom PHP monolith with:
- a front router in `index.php`
- a large AJAX/action handler in `pp-content/pp-include/pp-adapter.php`
- helper/service-style functions in `pp-content/pp-include/pp-functions.php`
- schema in `pp-content/pp-install/db.sql`

This matters for Android integration: your companion app must talk to `pp-adapter.php` actions (`action-companion`) rather than REST routes.

---

## 2. Backend Architecture

### Core request surfaces
1. **Public/API router**: `index.php`
   - Handles payment pages, checkout, API-key based merchant API, cron, webhooks.
2. **Admin + AJAX + Companion action router**: `pp-adapter.php`
   - Handles normal admin AJAX (`action`) and mobile companion calls (`action-companion`).
3. **Utility layer**: `pp-functions.php`
   - DB wrappers, auth header parsing, sender parsing, amount math, webhook sender, reconciliation.

### Routing style
- Path routing is hand-written using URL segments in `index.php`.
- Companion app routing is POST action switch-style in `pp-adapter.php` via `action-companion`.

---

## 3. Authentication System

## A) Merchant API key auth (for checkout API)
- `index.php` reads Authorization header via `getAuthorizationHeader()`.
- Validates key in `pp_api` table (`status='active'`, not expired).
- Validates scope in `api_scopes` JSON-like text (e.g., `create_payment`).

**Format expected:**
- Header: `Authorization: <api_key>` (not necessarily Bearer).

## B) Admin session + CSRF auth
- Admin AJAX actions require session login + CSRF token validation.

## C) Companion HMAC token path (non-session)
- If POST includes `pp-token`, backend skips CSRF and validates:
  - `pp-app-id`
  - `pp-app-timestamp`
  - `pp-token = HMAC_SHA256("<pp-app-id>|<pp-app-timestamp>", static_secret)`
- Static secret currently hardcoded in source (security risk).

## D) Companion device token auth
- During companion login, OTP is exchanged for long-lived device token (new `otp` value in `pp_device`).
- Subsequent companion calls authenticate by POST `token=<device otp>` and device must be `status='used'`.

---

## 4. Device Binding Flow

1. Admin opens Devices page and clicks connect.
2. Frontend calls `action=device-connect-info` (admin-authenticated).
3. Backend creates/updates `pp_device` row with:
   - `status='processing'`
   - random `otp`
   - `d_id` = current admin browser cookie token.
4. Admin UI renders QR with OTP.
5. Android app scans QR and calls `action-companion=login` + `onetimepassword=<otp>` + phone metadata.
6. If OTP exists, backend rotates OTP and marks device `status='used'`, stores device metadata, returns `token`.

**Important behavior:** token rotation at login invalidates QR OTP and produces runtime token.

---

## 5. QR Code Flow
- QR is generated client-side in admin (`qrcode.min.js`) from OTP returned by `device-connect-info`.
- QR payload is effectively the one-time password (or content containing it).
- No asymmetric signature on QR payload itself.
- Security relies on OTP secrecy + short practical lifetime.

---

## 6. API Endpoint List

## A) Merchant checkout API (`index.php`)
| Method/Path | Auth | Purpose |
|---|---|---|
| `POST /api/checkout/redirect` | `Authorization: <api_key>` | Create hosted checkout transaction |

Core validations:
- JSON body required.
- `return_url` and `webhook_url` domains must exist in `pp_domain` and be active.
- Scope must include `create_payment`.

## B) Companion endpoints (`pp-adapter.php`, POST)
All use form-urlencoded POST to adapter entrypoint with `action-companion`:

| action-companion | Required fields | Purpose |
|---|---|---|
| `login` | `onetimepassword`, `name`, `model`, `android_level`, `app_version` | Bind device using QR OTP; returns persistent token |
| `account-information` | `token` | Device/account info + grouped SMS records |
| `sms-transmit-bulk` | `token`, `sms_list` (JSON array) | Upload SMS list, parse/store, dedupe, balance checks |
| `sms-transmit-sender` | `token` | Returns sender whitelist definitions |
| `delete-sms-data` | `token`, `stored`, `used`, `error` | Cleanup synced SMS buckets |

---

## 7. Required Android App Features
1. QR scanner for OTP onboarding.
2. Device registration call (`action-companion=login`).
3. Secure token storage (Android Keystore).
4. SMS read permissions + background collection.
5. Sender filtering or all-SMS capture + server-side parsing fallback.
6. Batch upload (`sms-transmit-bulk`) with retry/backoff.
7. Sender metadata sync (`sms-transmit-sender`).
8. Local status dashboard from `account-information`.
9. Delete/cleanup operation after successful sync.

---

## 8. SMS Sync Architecture

### Ingestion path
- App posts `sms_list` JSON; each item contains roughly:
  - `id`, `sender`, `message`, `simSlot`, `timestamp`.
- Server resolves sender via `senderWhitelist()`.
- Server parses message via `MFSMessageVerified(sender_key, message)`.

### Classification
- `approved`: parsed + valid.
- `awaiting-review`: balance verification mismatch / SIM mismatch.
- `error`: unparseable/duplicate/invalid.
- `used`: later consumed by transaction matching flow.

### Deduplication key
- primary duplicate check uses (`sender_key`, `trx_id`) in `pp_sms_data`.

---

## 9. Transaction Processing Flow
1. Checkout transaction created (status initiated/pending).
2. SMS arrives from companion, parsed into `pp_sms_data`.
3. Cron (`/<cronPath>/<secret>`) runs periodic reconciliation:
   - finds pending transactions
   - matches against approved sms by sender_key/type/trx_id
   - tolerance checks with brand settings
   - marks transaction completed
   - marks SMS as used
4. Webhook dispatch is queued in `pp_webhook_log` and retried until success/limit.

---

## 10. Database Schema Explanation

## Critical tables for companion
- `pp_device`: binding state, OTP token, device metadata, sync heartbeat.
- `pp_sms_data`: parsed SMS ledger + review state.
- `pp_balance_verification`: expected running balance per sender/type/sim.
- `pp_transaction`: payment records waiting for SMS proof or gateway verification.
- `pp_webhook_log`: async webhook delivery queue.
- `pp_api`: merchant API keys/scopes.
- `pp_domain`: whitelist for return/webhook domains.

---

## 11. Security Model

### Confirmed
- API key auth with scope checks.
- OTP-based device bootstrap.
- Token rotation on companion login.
- CSRF for admin browser actions.
- Optional app-HMAC request signature path (`pp-token`).

### Risks / weaknesses
- Hardcoded HMAC secret in source.
- Device token is stored as plain DB OTP field (not hashed).
- No obvious token expiry/refresh for companion token.
- Potential SQL injection risk in some string-concatenated query areas (outside companion path too).

---

## 12. Missing Components / Closed Source Assumptions

Because this repo is monolithic PHP (not Laravel), missing requested components are:
- `routes/api.php`, `routes/web.php`, middleware classes, controllers, Eloquent models, migrations, queue workers.

Assumptions inferred:
- Companion app expected transport is standard HTTPS POST form-data/urlencoded to adapter endpoint.
- Polling model over REST-like calls; no websocket/socket backend found.
- QR payload likely contains raw OTP or OTP wrapper text.

---

## 13. Reverse Engineered Android API Specification

## Base
- `POST https://<host>/pp-content/pp-include/pp-adapter.php` (or equivalent routed PHP entry used by web app).
- Content-Type: `application/x-www-form-urlencoded`.

## 1) Login (bind)
Request:
```text
action-companion=login
onetimepassword=123456789012345
name=Pixel 7
model=Google Pixel 7
android_level=34
app_version=1.0.0
```
Response success:
```json
{ "status": "true", "token": "<device_token>" }
```

## 2) Account info
Request:
```text
action-companion=account-information
token=<device_token>
```
Response includes counts and arrays: `stored`, `used`, `error`.

## 3) Transmit bulk SMS
Request:
```text
action-companion=sms-transmit-bulk
token=<device_token>
sms_list=[{"id":"...","sender":"bkash","message":"...","simSlot":"1","timestamp":"..."}]
```

## 4) Fetch sender catalog
Request:
```text
action-companion=sms-transmit-sender
token=<device_token>
```

## 5) Delete synced data categories
Request:
```text
action-companion=delete-sms-data
token=<device_token>
stored=yes
used=no
error=yes
```

---

## 14. Suggested Android App Architecture
- **Data layer**: Retrofit/OkHttp + Kotlin serialization.
- **Domain layer**: use-cases for bind, upload, sync-state, cleanup.
- **Background**: WorkManager periodic workers (SMS upload + account sync).
- **Storage**: Room for local SMS staging + sync cursor; encrypted prefs for token.
- **Security**: Keystore-backed token encryption + cert pinning optional.

---

## 15. Suggested Tech Stack
- Kotlin + Coroutines + Flow
- Retrofit + OkHttp
- Room DB
- WorkManager
- ML Kit / ZXing for QR scanning
- Timber/Logcat + Crashlytics (optional)

---

## 16. Step-by-Step Android App Implementation Plan
1. Implement QR scan screen.
2. Build `login` call and persist token.
3. Add sender catalog sync (`sms-transmit-sender`).
4. Build SMS reader module + normalization.
5. Implement `sms-transmit-bulk` batching (e.g., 20–100 messages).
6. Add response-aware retry/backoff and duplicate suppression.
7. Implement dashboard via `account-information`.
8. Add cleanup API call policy after successful confirmed sync.
9. Add diagnostics screen (last sync, last error, token validity check).
10. Run end-to-end test with cron enabled server-side.

---

## 17. Sample API Requests/Responses

### Invalid token sample
```json
{ "status": "false", "title": "Authentication Failed", "message": "Please try again or scan the QR code again." }
```

### SMS success sample
```json
{ "status": "true", "title": "SMS Data Created", "message": "The sms data has been created successfully." }
```

### Duplicate tx sample
```json
{ "status": "false", "title": "Duplicate Transaction", "message": "The provided Transaction ID already exists in our system." }
```

---

## 18. Error Handling Strategy
- Treat all `status:"false"` as recoverable except invalid token.
- On auth failure: force rebind flow (scan QR again).
- On parsing errors: keep local copy, retry once, then quarantine.
- On duplicate: mark local as synced/ignored.
- Add idempotency locally using hash(sender+message+timestamp).

---

## 19. Local Testing Workflow
1. Install PipraPay and create admin account.
2. Open Devices → Connect and capture OTP/QR.
3. Use Postman/cURL to call `action-companion=login`.
4. Save returned token.
5. Send `sms-transmit-sender` and confirm sender catalog.
6. Send synthetic `sms-transmit-bulk` entries for known gateways.
7. Run cron endpoint and verify transaction status transitions + webhook logs.
8. Validate `account-information` counters.

---

## 20. Production Deployment Notes
- Enforce HTTPS only.
- Move hardcoded HMAC secret to environment/config.
- Add token expiration + refresh or rotate periodically.
- Add rate limiting on companion endpoints.
- Log companion device IP + user-agent anomalies.
- Back up `pp_sms_data`, `pp_transaction`, `pp_device` frequently.
- Monitor cron health and webhook retry queue depth.

---

## Assumptions vs Confirmed

### Confirmed from code
- Companion flow uses `action-companion` with the five actions above.
- Device bootstrap uses OTP from admin-generated device connection process.
- SMS parsing and balance verification feed transaction automation.
- Merchant checkout API key path is separate and header-authenticated.

### Assumed/inferred
- Exact QR payload string format (UI code suggests OTP-centric payload).
- Exact public URL path for posting companion requests may vary by deployment rewrite rules.
- No socket/webhook push to app; model appears pull/push over periodic HTTP.
