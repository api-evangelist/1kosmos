---
name: Verify a government ID document with IDVerify
description: Create a 1Kosmos IDVerify document-share session, hand the user the capture URL, and poll for the verification result.
api: openapi/1kosmos-blockid-openapi.yml
operations: [idVerificationCreateIDVerificationSession, idVerificationPollIDVerificationSession]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + https://developer.1kosmos.com/devportal/docs/idverify/dl/
---

# Verify a government ID document

Runs a driver license, passport, national ID card or SSN check through 1Kosmos IDVerify.

## Prerequisite

Complete `1kosmos-bootstrap-tenant.md` first. You additionally need the **IDVerify API key**
(`dvcID`) from the developer dashboard — it is not the license key.

## Steps

1. **Create the session** — `idVerificationCreateIDVerificationSession`
   `POST {docuverify}/document_share_session/create` with the standard `requestid` / `licensekey` /
   `publickey` headers and a body of the shape
   `{"data": {"dvcID": "<dvcID>", "sessionRequest": {"tenantDNS": "<dns>", ...}}}`.
   The response carries a `sessionId` and a capture `url` on the 1Kosmos capture host.

2. **Send the user to the capture URL.** They scan the document front/back and, depending on the
   configured flow, take a selfie for face-compare and liveness. Progress emits the `E_IDV_*` events
   catalogued in `asyncapi/1kosmos-idverify-events.yml`.

3. **Poll for the result** — `idVerificationPollIDVerificationSession`
   `POST {docuverify}/v3/document_share_session/result` with
   `{"data": {"dvcID": "<dvcID>", "sessionId": "<sessionId>"}}`.

4. **Read the outcome.** Session status is one of `Pending`, `In Progress`, `Completed`, `Expired`,
   `Declined`. Verification status is one of `Passed`, `Failed`, `Not performed`. Stop polling on
   `Completed`, `Expired` or `Declined`.

## Rules

- **Poll, do not subscribe.** There is no public webhook for verification completion — the result
  endpoint is the only documented delivery mechanism.
- `400 {"code":400,"message":"dvcID is required"}` and
  `400 {"code":400,"message":"dvcID is not allowed to be empty"}` both mean the IDVerify API key is
  missing from `data`, not from a header.
- **PII retention is configurable, not automatic.** `pii_retention_off` and `pii_retention_ttl`
  control whether captured document images are deleted immediately or held up to 7 days. Confirm the
  tenant setting before storing anything yourself.
- Never log the capture URL — it is a bearer link to the user's document capture flow.
