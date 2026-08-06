---
name: Run an IAL2 identity-proofing workflow
description: Start a 1Kosmos IAL2 identity-proofing journey, read per-node results, pull the result summary, and confirm the user's assurance level.
api: openapi/1kosmos-blockid-openapi.yml
operations: [ial2CreateWorkflow, ial2ResultSummary, workflowAPICreateWorkflowInstance, workflowAPINodeResult, ialFetchIALOfAUser]
generated: '2026-08-05'
method: generated
source: openapi/1kosmos-blockid-openapi.yml + https://docs.1kosmos.com/productdocs/docs/idverify/ial2-verification/oauth2-oidc/
---

# Run an IAL2 identity-proofing workflow

A workflow chains the evidence a NIST SP 800-63-3 IAL2 proofing requires — liveness, two documents,
an SSN check — into one journey. 1Kosmos is Kantara-approved at IAL2/AAL2, so this is the flow that
carries that assurance.

## Prerequisite

Complete `1kosmos-bootstrap-tenant.md`. The proofing service is `idproofing` /`idproofingapi` in the
service-discovery document.

## Steps

1. **Start the journey** — `ial2CreateWorkflow`
   `POST https://{dns}/idproofingapi/workflow/workflow_instance/create` with
   `{"data": {"workflowId": "<shortid>-<flow name>", "metaData": {"firstname": "…", …}}}`.
   A published example `workflowId` has the form `586ea2ee-liveid_2doc_ssn` — liveness + two
   documents + SSN. Keep the returned instance id.

   The tenant/community-scoped variant is `workflowAPICreateWorkflowInstance`
   (`POST {wf_api}/workflowapi/workflow_instance/tenant/{tenantId}/community/{communityId}/create`),
   which takes an **encrypted** body: `{"data": "<encrypted payload>"}`.

2. **Watch individual steps** — `workflowAPINodeResult`
   `GET {wf_api}/workflowapi/workflow_instance/{wfInstanceId}/tenant/{tenantId}/community/{communityId}/node/{nodeId}`
   returns the result of one node. Use this to show progress or to identify which evidence failed.

3. **Read the outcome** — `ial2ResultSummary`
   `GET https://{dns}/idproofingapi/workflow/workflow_instance/{instanceId}/result_summary`.

4. **Confirm the assurance level** — `ialFetchIALOfAUser`
   `GET https://{dns}/api/r1/community/{communityName}/userid/{userId}/ial` returns the identity
   assurance level(s) the user now holds, and the evidence behind them. Requires `X-TenantTag`,
   `tenant_name` and `community_name` headers alongside the usual three.

## Rules

- **This is asynchronous and human-paced.** The user is scanning documents on a phone. Poll the
  result summary on a backoff; there is no completion webhook.
- **Step 4 is the authoritative check.** Do not infer assurance from a workflow node's result —
  read the user's IAL.
- Workflow definitions are configured per tenant in AdminX ("Build Your Own Intuitive Workflow for
  Verification Journeys", 1.12.04.01). A `workflowId` from one tenant will not exist in another.
- Everything in `metaData` is PII. Apply the tenant's `pii_retention_off` / `pii_retention_ttl`
  policy to anything you keep.
