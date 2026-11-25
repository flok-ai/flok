# ADR-017: Caller authentication for platform services — Kubernetes ServiceAccount tokens, one step record per pod

- **Status:** Accepted — not built yet
- **Date:** 2026-11-25
- **Context:** How agent pods and trusted workflow steps authenticate to platform services, and how step records relate to pods. Supersedes [ADR-016](./016-hmac-caller-authentication.md).

## Context

- [ADR-016](./016-hmac-caller-authentication.md) chose HMAC tokens for the first release as a pragmatic choice and deferred Kubernetes ServiceAccount tokens, to revisit "once the broker is built". The GitHub token broker is built and live-verified, with the same mechanism this ADR adopts ([token broker](../reference/token-broker.md)).
- Today a **step record** is created per role when the run is created: exactly one `planner` and one `coder`. The trusted `foreman` step receives one HMAC token per role and writes one Kubernetes Secret per role. Every pod that mounts a Secret is that step to platform services.
- Loops break that model. [04-workflow](../04-workflow.md) unrolls every bounded loop into a DAG where **each iteration is a new pod**: a fix round after review, retries, review–fix cycles. Under HMAC a second coder pod can only reuse the first coder's credential.
- HMAC tokens expire a fixed time after **run creation**, so late steps (fix rounds, transcript uploads, usage reports) run into a ceiling unrelated to their own duration.

## Decision

**Platform services authenticate callers with projected Kubernetes ServiceAccount tokens, validated with TokenReview, as the broker does. Every pod gets its own credential and its own step record; no credential is shared between pods.**

The requirement is **no credential reuse**, not a separate identity per attempt: a second coder pod has exactly the first coder's permissions (same role) but its own credential. Removing the run-creation expiry ceiling is a side effect.

### Target design

- **Credential:** each pod mounts a projected ServiceAccount token with audience `platform-services`, separate from the broker's `token-broker` audience so neither service can replay the other's token. The kubelet refreshes it; the pod re-reads the file per request.
- **Verification:** TokenReview (authenticated, audience) → namespace and ServiceAccount from the username, pod name and UID from the bound-pod claims → look up the pod and check UID and ServiceAccount, as the broker does.
- **Role:** the pod's ServiceAccount. Permissions stay role-based.
- **Run:** found from the pod's owner reference, the Workflow UID. **No run → reject.** Agent pods never create runs.
- **Step record per pod:** created on the pod's **first authenticated call**, keyed by run and Argo node ID. Retries and loop iterations get their own records because Argo gives every pod its own node. The workflow's exit handler reports on the same key, so pods that fail before calling platform services also appear.
- **`foreman` stays the first step of every workflow.** It authenticates with its own ServiceAccount token, and `foreman` is the only role that may create runs. It registers the run with its **workflow type, repository, issue and per-step models**. The workflow type comes from the Workflow's `workflows.argoproj.io/workflow-template` label, which the Argo controller sets from `spec.workflowTemplateRef` and which a submitter can't override.
- **Policy per workflow type:** platform services decide which roles may take part in a run of that type and which artifacts each role reads and produces. New workflow types are policy entries, not new authentication code.
- **Artifacts per record:** artifacts are unique per producing record and name, so each pod has its own `transcript` without a schema change.

### MVP / POC scope

- Built as one hard switch across platform services, workflows and infrastructure: planner and coder move to ServiceAccount tokens in the same change, with no period that accepts both. Runs are submitted by hand, so none are submitted during the switch.
- Bindings stay derived from the role while each workflow has one producer per artifact: one `planner` produces `plan`; any `coder` or `reviewer` record reads the run's `plan`. The check that a run has exactly one planner and one coder becomes "exactly one planner, any number of the other roles".
- Reuse from the broker: the Kubernetes adapters (TokenReview and bound-pod lookup with strict parsing) and the authorization pattern, ported to platform services with the audience as a parameter.
- Removed: per-step HMAC tokens, the run-creation credential response, the step-token key in Key Vault, the HMAC Secrets per role, and `foreman`'s permission to create Secrets.
- Platform services gain `tokenreviews: create` and `pods: get` in `agents`, the same pair the broker has.
- Not in scope: artifacts with several versions in one run (loops whose iterations produce the same artifact); per-type policy beyond the current workflow; callers outside the cluster.

## Alternatives considered

| Alternative | Status |
|---|---|
| Keep HMAC, one record per role, later pods reuse the role's credential | Rejected: credential reuse; attempts merge into one record (cost, transcript, failure per attempt lost). |
| HMAC with a fixed list of records per run (for example `coder-fix-1` to `coder-fix-3`) | Rejected: works only for a known maximum, and every loop shape changes run creation. |
| HMAC with records created on demand by `foreman` before each attempt | Rejected: per-pod credentials are possible, but a trusted pod runs before every attempt, `foreman` keeps creating Secrets, and the expiry ceiling stays. |
| Pull exchange: the pod presents its ServiceAccount token, platform services issue a scoped HMAC token (ADR-016 Open) | Rejected: needs all the ServiceAccount verification anyway and keeps HMAC on top. |

## Consequences

### Positive

- No credential reuse: one credential per pod, bound to that pod and useless once it's gone.
- Pods scale: loops, retries and fix rounds get their own records, transcripts, usage and failures without changes to run creation.
- No run-creation expiry ceiling for late steps.
- Fewer secrets and privileges: no HMAC key to store and rotate, no Secrets per role, no Secret creation for `foreman`.
- One authentication mechanism for the broker and platform services, with shared code.

### Accepted trade-offs

- **No callers outside the cluster.** Local development and tests use a mock token validator, as the broker's tests do. The requirement for cluster-free callers still exists and is deferred, not removed.
- Platform services depend on the Kubernetes API for every request (TokenReview and pod lookup), and gain read access to pods in `agents`, in addition to their Blob and database access.
- Whoever submits a Workflow still chooses its template, parameters and `entrypoint`. Accepted while only allowlisted repositories without outside triggers are used; the declarative repository configuration ([flok#39](https://github.com/flok-ai/flok/issues/39)) removes Argo from the submitter's view.

## Open

- Artifact bindings when one run produces several versions of an artifact (which `plan` or which coder output does attempt 2 read).
- Who reports the PR URL and per-step durations: the exit handler, the step's own reports, or both.
- Whether the broker should also require an existing run, so a Workflow submitted with an `entrypoint` that skips `foreman` gets no GitHub token.
- Callers outside the cluster, when needed again.

## Verification

To do, on the next run, before implementation relies on it (Argo v4.1.6, while the Workflow still exists):

- The Workflow carries `workflows.argoproj.io/workflow-template` with the template name. Confirmed in the Argo source (`setWfTemplateLabel` in `workflow/controller/operator.go` at tag `v4.1.6`); not yet observed in the cluster.
- Agent pods have an owner reference to the Workflow with its UID.
- Agent pods carry a per-pod node ID annotation, and a retried pod gets a new one.
- Which of these a template author can override.
