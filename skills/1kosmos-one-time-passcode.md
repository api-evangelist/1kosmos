---
name: Send and verify a one-time passcode
description: Generate a BlockID one-time passcode over SMS, email or voice and verify the code the user enters.
api: openapi/1kosmos-blockid-openapi.yml
operations: [otpGenerateOTPSMS, otpVerifyOTPR3, otpVerifyOTPR2]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + https://developer.1kosmos.com/devportal/docs/otp/request/
---

# Send and verify a one-time passcode

## Prerequisite

Complete `1kosmos-bootstrap-tenant.md`. You need `tenantId`, `communityId` and the encrypted
`licensekey`.

## Steps

1. **Generate** — `otpGenerateOTPSMS`
   `POST {otp-host}/api/r2/otp/generate`. One operation, three delivery channels, chosen by which
   destination fields you send:
   - SMS: `{"userId": "...", "communityId": "...", "tenantId": "...", "smsTo": "...", "smsISDCode": "..."}`
   - Email: `{"userId": "...", "communityId": "...", "tenantId": "...", "emailTo": "...", "emailFrom": "..."}`
   - Voice: `{"userId": "...", "communityId": "...", "tenantId": "...", "voiceTo": "...", "voiceISDCode": "..."}`

   A success returns **202** with `{"messageId": "<uuid>", "info": "OTP request accepted"}`.

2. **Verify** — `otpVerifyOTPR3`
   `POST {otp-host}/api/r3/otp/verify` with
   `{"code": "<user input>", "userId": "...", "communityId": "...", "tenantId": "..."}`.
   Use **r3**: the published collection describes it as the standardized verify endpoint.
   `otpVerifyOTPR2` (`/api/r2/otp/verify`) is the older revision and is still live — call it only
   for a legacy integration.

## Rules

- **A wrong code is a 404, not a 401.** `{"error_code": 404, "message": "Invalid OTP", "status":
  false}`. Do not treat 404 as "endpoint missing" on this operation.
- **403 `OTP locked for the user`** means the per-user OTP rate limit (AdminX 1.12.08.03) has tripped.
  Back off; do not retry the generate call in a loop.
- **404 `Unable to load tenant/community`** on generate means `dns` + `communityName` did not resolve
   — re-run the bootstrap skill.
- **No idempotency key.** Calling generate twice sends two passcodes and invalidates your UX. Guard
  the call on your side.
- The `noecdsa` header exists on this surface to skip license-key encryption. Prefer the encrypted
  path in production.
