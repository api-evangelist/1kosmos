---
name: Issue and verify W3C Verifiable Credentials
description: Create a Verifiable Credential from a payload, check its status, bundle credentials into a Verifiable Presentation, and verify either one.
api: openapi/1kosmos-blockid-openapi.yml
operations: [verifiableCredentialsCreateVerifiableCredentialFromPayload, verifiableCredentialsGetVerifiableCredentialStatus, verifiableCredentialsVerifyVerifiableCredential, verifiableCredentialsIssueVerifiablePresentation, verifiableCredentialsVerifyVerifiablePresentation]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + https://developer.1kosmos.com/devportal/docs/verifiable-credentials/overview/
---

# Issue and verify W3C Verifiable Credentials

BlockID issues credentials in the W3C Verifiable Credentials data model
(`"@context": ["https://www.w3.org/2018/credentials/v1", …]`). Documented credential types include
driver licenses and employment cards.

## Prerequisite

Complete `1kosmos-bootstrap-tenant.md`. Every path here is scoped
`/vcs/tenant/{tenantID}/community/{communityID}/…` and every call needs `licensekey`, `publicKey`
and `requestid`.

## Issue

1. **Create a credential** — `verifiableCredentialsCreateVerifiableCredentialFromPayload`
   `POST /vcs/tenant/{tenantID}/community/{communityID}/vc/from/payload/{vcType}` with an `info`
   object carrying the subject claims (`id`, `firstName`, `lastName`, `companyName`, …).
   `{vcType}` selects the credential type.

2. **Check status** — `verifiableCredentialsGetVerifiableCredentialStatus`
   `GET /vcs/tenant/{tenantID}/community/{communityID}/vc/{vcID}/status`.
   `404 {"code":404,"message":"VC not found"}` when the id is wrong or the credential was removed.

## Present

3. **Bundle into a presentation** — `verifiableCredentialsIssueVerifiablePresentation`
   `POST /vcs/tenant/{tenantID}/community/{communityID}/vp/create` with `{"vcs": [{"vc": {…}}, …]}`.
   `400 {"code":400,"message":"Attribute does not exist"}` means a requested claim is not on the
   credential — check the claim names before building the presentation.

## Verify

4. **Verify a credential** — `verifiableCredentialsVerifyVerifiableCredential`
   `POST /vcs/tenant/{tenantID}/community/{communityID}/vc/verify` with `{"vc": {…}}`.
5. **Verify a presentation** — `verifiableCredentialsVerifyVerifiablePresentation`
   `POST /vcs/tenant/{tenantID}/community/{communityID}/vp/verify` with `{"vp": {…}}`.

## Rules

- **The whole credential goes in the body.** These verify endpoints take the full VC/VP JSON-LD
  document, not an identifier.
- `401 {"code":401,"message":"Unauthorized"}` on any of these five is a license-key problem, not a
  credential problem.
- A payload whose validity window has passed fails at issue time with
  `400 "info failed custom validation because Expired payload"`.
- `500 {"code":500,"message":"Internal Server Error"}` is a documented outcome of the credential
  verify endpoint — treat it as retryable, but only with backoff.
- No idempotency key: re-posting the same payload mints a second credential.
