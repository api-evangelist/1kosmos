---
name: Bootstrap a 1Kosmos BlockID tenant session
description: Resolve a BlockID tenant and community, discover its microservice hosts, and build the encrypted licensekey/publickey headers every other BlockID call requires.
api: openapi/1kosmos-blockid-openapi.yml
operations: [setUpGetTenantCommunityInfo, setUpFetchSD]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + conventions/1kosmos-conventions.yml
---

# Bootstrap a BlockID tenant session

Every other 1Kosmos skill depends on this one. There is no `api.1kosmos.com` and no bearer token:
you address a **tenant** by its DNS name, resolve its **community**, discover the per-service hosts,
and sign each request with an ECDSA-encrypted license key.

## You need

Four values, all issued in the BlockID developer dashboard
(https://developer.1kosmos.com/devportal/dashboard/):

- `dns` — tenant DNS name, e.g. `blockid-trial.1kosmos.net`. This is also the API host.
- `communityName`
- `licenseKey`
- your own ECDSA key pair (the SDKs generate one; `publickey` is sent, the private key never leaves you)

## Steps

1. **Resolve the tenant and community** — `setUpGetTenantCommunityInfo`
   `POST https://{dns}/api/r1/system/community_info/fetch` with body
   `{"dns": "<dns>", "communityName": "<communityName>", "skipLogos": false}`.
   Keep from the response: `community.id` (→ `communityId`), `community.tenantid` (→ `tenantId`),
   `tenant.tenanttag` (→ the `X-TenantTag` header) and the **community public key**.

2. **Derive the shared secret.** Combine your ECDSA private key with the community public key
   (`BIDECDSA` in every first-party helper SDK). Everything encrypted below uses this secret.

3. **Discover the service hosts** — `setUpFetchSD`
   `GET https://{dns}/caas/sd`. The response maps service keys to base URLs — `reports`, `otp`,
   `vcs`, `user_management`, `docuverify`, `sessions`, `webauthn`, `oauth2`, `acr`, `idproofing`.
   Read the host for the service you are about to call from here; do not hard-code it.

4. **Build the standard headers** for every subsequent call:
   - `licensekey` — your license key, encrypted with the shared secret
   - `publickey` — your ECDSA public key
   - `requestid` — an encrypted `{uuid, appid, timestamp}` object (request tracing)
   - `X-TenantTag` — required by the reports, user-management and access-code services
   - `Content-Type: application/json`
   - `noecdsa` — only on OTP and access-code calls when you deliberately skip encryption

## Rules

- **Cache the community info and the SD document.** Both are stable; re-fetching them on every call
  triples your request count.
- **There is no idempotency contract.** Retrying a create call creates a second object. See
  `conventions/1kosmos-conventions.yml`.
- If a call returns `400 {"errors":[{"param":"licensekey","message":"This header can't be
  decrypted"}]}`, the shared secret is stale — re-run step 1 and step 2.
