# PipraPay Android Companion API Examples

This file provides practical request/response examples for the PipraPay Companion integration.

## Base URL
Use your site root as endpoint:

```text
POST https://your-domain.com/
```

Companion actions are handled by `action-companion`.

Content type:

```text
application/x-www-form-urlencoded
```

---

## 1) Bind Device (QR OTP Login)

### cURL
```bash
curl -X POST "https://your-domain.com/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "action-companion=login" \
  --data-urlencode "onetimepassword=123456789012345" \
  --data-urlencode "name=Pixel 8" \
  --data-urlencode "model=Google Pixel 8" \
  --data-urlencode "android_level=34" \
  --data-urlencode "app_version=1.0.0"
```

### Success
```json
{
  "status": "true",
  "token": "A1B2C3D4E5F6G7"
}
```

### Failure
```json
{
  "status": "false",
  "title": "Invalid Credentials",
  "message": "Please enter the correct credentials or scan the QR code again."
}
```

---

## 2) Account Information

### cURL
```bash
curl -X POST "https://your-domain.com/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "action-companion=account-information" \
  --data-urlencode "token=A1B2C3D4E5F6G7"
```

### Success Example
```json
{
  "status": "true",
  "fullname": "John Doe",
  "email": "john@example.com",
  "stored_count": 2,
  "used_count": 1,
  "error_count": 0,
  "stored": [
    {
      "id": "1401",
      "sender": "bkash",
      "message": "Cash In Tk 500.00 from 017... TrxID: ABC123",
      "reason": "--",
      "simslot": "1",
      "timestamp": "May 07, 2026 10:20 AM",
      "status": "stored"
    }
  ],
  "used": [],
  "error": []
}
```

### Token Error
```json
{
  "status": "false",
  "title": "Authentication Failed",
  "message": "Please try again or scan the QR code again."
}
```

---

## 3) Fetch Sender Definitions

### cURL
```bash
curl -X POST "https://your-domain.com/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "action-companion=sms-transmit-sender" \
  --data-urlencode "token=A1B2C3D4E5F6G7"
```

### Success Example
```json
{
  "status": "true",
  "senders": [
    {
      "provider_key": "bkash",
      "name": "bKash",
      "currency": "BDT",
      "balance_verify": "true"
    },
    {
      "provider_key": "nagad",
      "name": "Nagad",
      "currency": "BDT",
      "balance_verify": "true"
    }
  ]
}
```

---

## 4) Send SMS in Bulk

### Request Payload Structure
`sms_list` is JSON (string field in form body), example item:

```json
{
  "id": "sms_local_001",
  "sender": "bkash",
  "message": "Cash In Tk 500.00 from 017xxxxxxx. Fee Tk 0.00. Balance Tk 2450.00. TrxID ABC123 at 07/05/2026 09:33",
  "simSlot": "1",
  "timestamp": "2026-05-07 09:33:00"
}
```

### cURL
```bash
curl -X POST "https://your-domain.com/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "action-companion=sms-transmit-bulk" \
  --data-urlencode "token=A1B2C3D4E5F6G7" \
  --data-urlencode 'sms_list=[{"id":"sms_local_001","sender":"bkash","message":"Cash In Tk 500.00 from 017xxxxxxx. Fee Tk 0.00. Balance Tk 2450.00. TrxID ABC123 at 07/05/2026 09:33","simSlot":"1","timestamp":"2026-05-07 09:33:00"}]'
```

### Success
```json
{
  "status": "true",
  "title": "SMS Data Created",
  "message": "The sms data has been created successfully."
}
```

### Duplicate
```json
{
  "status": "false",
  "title": "Duplicate Transaction",
  "message": "The provided Transaction ID already exists in our system."
}
```

### Invalid/Unknown Parse
```json
{
  "status": "false",
  "title": "Invalid or unknown MFS message",
  "message": "Please fill in all required fields before proceeding."
}
```

---

## 5) Delete SMS Data Buckets (Server-side cleanup)

Flags accepted: `stored`, `used`, `error` with values `yes` / `no`.

### cURL
```bash
curl -X POST "https://your-domain.com/" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "action-companion=delete-sms-data" \
  --data-urlencode "token=A1B2C3D4E5F6G7" \
  --data-urlencode "stored=yes" \
  --data-urlencode "used=yes" \
  --data-urlencode "error=no"
```

### Typical Success Example
```json
{
  "status": "true",
  "title": "Success",
  "message": "SMS data deleted successfully."
}
```

---

## Android Implementation Notes
- Endpoint is **root `/`**, not `/api/...` for companion calls.
- Always send `application/x-www-form-urlencoded`.
- Save `token` securely after bind; required for all subsequent calls.
- If server returns authentication failed, force QR re-bind.
