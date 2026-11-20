# flok

**Unsupervised coding agents in the cloud.** [flok](https://github.com/flok-ai) takes a scoped GitHub issue to a reviewed pull request — with real isolation boundaries, agents with their own identity, and an enforced multi-agent workflow. Built in public; this repo is its design and context layer.

![Two modes: an interactive session on my laptop writes scoped issues to GitHub; unsupervised agents in the cloud pull issues and open pull requests](./images/two-modes.svg)

**Pair & delegate.** Architecture and scoping stay interactive — me and a coding agent, on my laptop. Implementation runs headless in the cloud, one pod per agent step. GitHub issues are the handoff.

**100% open source.** Most open source agent tools run on a developer's laptop as wrappers around Codex or Claude Code, and the scalable ones keep key parts closed. flok is cloud-native and fully open: every component, from the infrastructure to the token broker, is in the open.

## What's here

This repo is where I design the platform and scope work. The components are built in their own repositories, where agents pick up the scoped tickets — without context from here, so a badly scoped ticket shows up as a failure.

- [Architecture](./architecture/) — how the platform works: use case, architecture, requirements, workflow
- [Decisions](./architecture/decisions/) — ADRs, one decision per file, target design vs. what the POC builds

### Components

Each component is built, versioned and deployed on its own.

| Repository | What it does |
|---|---|
| [agent-images](https://github.com/flok-ai/agent-images) | `agent-base` image: goose, runtime entrypoint, credential and signing setup |
| [github-broker](https://github.com/flok-ai/github-broker) | Just-in-time GitHub tokens, scoped to an agent's role and one repository |
| [infrastructure](https://github.com/flok-ai/infrastructure) | Terraform for Azure: AKS, workload identities, Key Vault, storage, PostgreSQL |
| [platform-services](https://github.com/flok-ai/platform-services) | Runs, HMAC step tokens and the artifact API |
| [workflows](https://github.com/flok-ai/workflows) | Argo workflow templates and agent recipes |

> [!IMPORTANT]
> This is _not_ a reference architecture. It's a portfolio piece and proof of concept I'm building out in public to share my learnings – and to put my money where my mouth is. Although a POC, I'm still applying 10+ years of cloud engineering experience to set up enough guardrails and enforced security boundaries that I will let agents work unsupervised for me.
>
> For most individuals this setup is overkill. For most organizations, however, as a former enterprise architect, I consider this to be _minimum_ due diligence.

## Project Status

- **Design** — [ADR-001 – 016](./architecture/decisions/) accepted
- **Dev container agent** — working: own GitHub account, GPG-signed commits, unattended runs after a confirmed read-back ([ADR-001](./architecture/decisions/001-dev-container-agent-identity.md))
- **goose in a container** — working with `docker run`: issue in, PR out
- **Cloud platform on AKS** — [cluster deployed](https://github.com/flok-ai/infrastructure), Argo Workflows installed, agent secrets from Key Vault through workload identity
- **Platform services** — [deployed in the cluster](https://github.com/flok-ai/platform-services): create a run, HMAC step tokens, artifact API with SAS URLs, Entra authentication to PostgreSQL
- **Workflow** — [`foreman → planner → coder`](https://github.com/flok-ai/workflows) runs end to end with `argo submit` (or `make submit`): issue in, signed PR out
- **GitHub token broker** — [in use by the workflow](https://github.com/flok-ai/github-broker): each agent pod gets a just-in-time GitHub token scoped to its role and one repository. PRs are opened by the GitHub App; commits stay signed by the agent. No shared GitHub token is in use.

### Milestones

- [x] **Initial release** (2026-11-11) — one issue end to end in the cluster: a planner writes a plan, a coder implements it and opens a signed PR, I review and merge.
- [x] **Just-in-time credentials** (2026-11-20) — every component runs on short-lived, scoped credentials; the shared GitHub token is retired.

**Next:** better issue scoping and agent output quality, and saving agent session transcripts ([#30](https://github.com/flok-ai/flok/issues/30)).

**Later:** reviewer step, webhook trigger, role images with skills baked in.

## What's New?

### 2026-11-20

**Changed**

- **Renamed to flok** — like a flock of geese: every agent runs goose. The repositories moved to the [flok-ai](https://github.com/flok-ai) organization, and pull requests are now opened by `flok-ai[bot]`.
- **Open source licenses:** components under AGPL-3.0-only, docs and diagrams under CC BY-SA 4.0, with attribution required (see [License](#license)). Releases [v0.1.0](https://github.com/flok-ai/flok/releases/tag/v0.1.0) (the POC, MIT) and [v0.2.0](https://github.com/flok-ai/flok/releases/tag/v0.2.0) (relicensed) pin each component.
- **Submodules removed:** each component is built, versioned and deployed on its own; see [Components](#components).

**Milestone**

- **Every component runs with just-in-time credentials.** Agent pods get their GitHub token from the broker at startup, scoped to their role and one repository and valid for an hour. The shared GitHub token is revoked and removed from the cluster and its Key Vault grants. The first run without it opened a PR as the GitHub App, with a **Verified** commit and green CI.

**Validated**

- **Infrastructure access re-checked, nothing left unverified.** Every workload identity holds only the secret-scoped grants it needs, nothing inherited. A pod can't mount another identity's secrets, agent pods still can't reach the cloud metadata endpoint, and logs stay within budget. See the [verification log](https://github.com/flok-ai/infrastructure/blob/main/docs/verification-log.md).

**Learned**

- **A bare registry name pulls from Docker Hub.** `aisdlc…cr` instead of `aisdlc…cr.azurecr.io` made the planner wait for an image that doesn't exist until its deadline. `make submit` now rejects it up front.

### 2026-11-18

**Validated**

- **GitHub token broker works in the cluster.** A pod presents its own Kubernetes token and gets a GitHub token that lasts an hour, limited to its role and one repository. The planner's push was denied, the coder's succeeded, and every wrong caller was refused: missing assignment, wrong role, wrong token audience, or a request trying to choose its own repository. No token reached the logs. See the [token broker reference](./architecture/reference/token-broker.md).

**Learned**

- **A GitHub App has two IDs that are easy to mix up.** The App ID comes from the App's own settings page; the installation ID belongs to the App's installation on a repository. With the wrong one, GitHub answers "Integration not found", and because the broker deliberately hides upstream errors, finding it took a direct call to GitHub.

### 2026-11-11

**Milestone**

- **Initial release works end to end.** One `argo submit` with a repository and issue number: a trusted `foreman` step creates the run, a planner writes and uploads a plan, a coder implements it and opens a signed PR. The first agent PR took under 2.5 minutes, showed **Verified** on GitHub, and I merged it.

**Decided**

- **Agents transfer artifacts themselves**, with `curl` commands in their instructions, rather than through wrapper code. Accepted trade-off: the model can read its own short-lived step token. The pod already holds stronger credentials, so hiding it would have added code without adding protection.
- **Agent instructions live in the workflows repo** next to the templates, for now. A practical choice for this release, not an architecture decision.

**Learned**

- `goose run` exits successfully even when the model call fails. Each step now checks that its result exists: an uploaded plan, an opened PR.
- Argo copies the agent container's volume mounts, including secrets, into its own helper container. The templates remove those copies.

### 2026-11-10

**Added**

- **Platform services deployed on AKS**: HMAC step tokens, creating a run, and a workload identity token for PostgreSQL. Migrations run at startup as the service's own identity. Verified from an agent pod: create a run, upload a plan, download it.
- **Agent image entrypoint**: loads the secrets mounted from Key Vault and sets up the Git identity and commit signing.

**Decided**

- **Agents read the long-lived secrets they need** through workload identity, one secret-scoped grant each. Replaces the signing-key-only exception. See [ADR-003](./architecture/decisions/003-platform-services-vs-agent-pods.md#workload-identity-for-long-lived-secrets).
- **Step tokens expire after 15 minutes** as a cost cap: a slow run fails rather than keeps spending.

### 2026-11-05

**Decided**

- **Initial release scope:** one issue end to end in the cluster. Start a run with the Argo CLI; a planner writes a plan, a coder implements it and opens a PR, and I review and merge. No reviewer step yet.
- **HMAC tokens for platform services**, reusing my Tally Split implementation, so callers work without a cluster. Kubernetes ServiceAccount authentication is deferred. See [ADR-016](./architecture/decisions/016-hmac-caller-authentication.md).
- **One exception to "agents have no cloud identity":** the coder can read only its commit-signing key from Key Vault. See [ADR-003](./architecture/decisions/003-platform-services-vs-agent-pods.md#exception-workload-identity-for-the-commit-signing-key).
- **Shared agent instructions** now prefer the smallest implementation that meets the acceptance criteria, after agents kept proposing production-grade machinery for a POC.

**Added**

- **Argo Workflows installed** on AKS with Helm. See [Infrastructure](https://github.com/flok-ai/infrastructure).
- **Trusted `foreman` step:** its own identity and RBAC to create a run and hand each agent step a scoped token. Platform services consolidated into one identity.
- **Key Vault secrets created** by script: platform-service tokens and the agent's GPG signing key.

### 2026-11-04

**Validated**

- **Artifact SAS URLs** signed with user delegation keys, no storage account keys. Upload, download and overwrite work against Azure; wrong permissions and other containers return 403. See [ADR-015](./architecture/decisions/015-sas-signing-and-local-identity.md).

### 2026-11-01

**Added**

- **Artifact API** in [Platform services](https://github.com/flok-ai/platform-services): register, upload and download artifacts through scoped SAS URLs, backed by PostgreSQL. Plus logging, a readiness endpoint and an API docs site.

**Changed**

- Removed an unneeded service layer from platform services in favor of explicit guards and plain functions.

### 2026-10-27

**Added**

- **Cost control:** GitHub workflow stops AKS and PostgreSQL manually and daily at 8pm.

### 2026-10-25

**Decided**

- **Commit signing:** native Git with GPG, one key per agent role. See [ADR-014](./architecture/decisions/014-commit-signing.md).

**Added**

- [GitHub token broker](https://github.com/flok-ai/github-broker) bootstrapped, and the [workflows](https://github.com/flok-ai/workflows) repo created.

**Validated**

- **Agent pod restrictions:** agent pods can't reach the cloud metadata endpoint (IMDS), and privileged pods such as `hostNetwork` are rejected.

### 2026-10-22

**Added**

- **Azure infrastructure deployed:** AKS cluster, networking, shared services and workload identities. See [Infrastructure](https://github.com/flok-ai/infrastructure).
- **Agent base image built:** Goose on Debian, with Python and Node, running as a non-root user. AMD64 build and offline smoke checks passed. See [Agent images](https://github.com/flok-ai/agent-images).

**Decided**

- Use GitHub CLI for GitHub operations and keep Goose a light wrapper around capable models, prioritizing API-token cost efficiency. See [Image ADRs](https://github.com/flok-ai/agent-images/tree/main/architecture/decisions).

### 2026-10-20

**Design**

- Established the platform architecture and ADRs covering isolation, Argo orchestration, agent identity, credentials and completion gates. See [Architecture](./architecture/).

**Validated**

- Dev container agents commit under their own GitHub identity with GPG signatures. See [ADR-001](./architecture/decisions/001-dev-container-agent-identity.md)

## Resources

Inspired by

- [Build Systems, Not Code - Angie Jones, Agentic AI Foundation](https://www.youtube.com/watch?v=ZD9-4fW2HhM) AI Engineer, YouTube, 25 June 2026
- [Building an Autonomous Engineering Org - Angie Jones, Agentic AI Foundation](https://youtu.be/whue9_YquGA?si=7K0xn9DqN3oSDtUT) – AI Engineer, YouTube, 28 June 2026
- [Agents Without Code: Skills, YAML, and Filesystems Replaced Python — Philipp Schmid, Google DeepMind](https://www.youtube.com/watch?v=fjF8EKnxKCU) – AI Engineer, YouTube, 28 June 2026
- [Why Software Factories Fail – Dex Horthy, Human Layer](https://www.youtube.com/watch?v=Ib5GBkD555M)  AI Engineer, YouTube, 23 July 2026
- [The State of AI in Software Development: Data from 400+ Orgs — Justin Reock, DX](https://www.youtube.com/watch?v=Se8jHLliLXE) – AI Engineer, YouTube, 30 Sept. 2026.
- [12 Factor Agents - Principles for building reliable LLM applications](https://github.com/humanlayer/12-factor-agents) Human Layer, 2025.

## License

Docs, architecture decision records and diagrams in this repository are licensed under [CC BY-SA 4.0](./LICENSE); scripts under AGPL-3.0-only (see [REUSE.toml](./REUSE.toml)). The flok components are AGPL-3.0-only in their own repositories. Reuse requires attribution as described in [NOTICE](./NOTICE); GitHub's "Cite this repository" uses [CITATION.cff](./CITATION.cff).
