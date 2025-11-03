# AI-Native VCS (ANVCS)

> **SPEC REPOSITORY — DRAFT STATUS**
>
> This repo publishes the **ANVCS specification** and design documents. It **may** also include an **optional, non‑production reference implementation** to prove out the spec. The spec is **DRAFT** (not Candidate, not Stable) and may change.

*AI-Native Version Control — CRDT live editing + DAG commits + Git-compatible. Offline-first, multi-agent ready, **cloud-agnostic versioned artifacts** via pluggable adapters.*

---

## At a glance — what this spec addresses

> This repo hosts the **ANVCS** spec and the reference implementation plan for a Git-compatible, CRDT-powered, AI-native VCS that:
>
> * lets **agents and humans collaborate live** without conflict storms,
> * preserves **Git’s immutability** and CI/CD workflows,
> * works **offline/air-gapped** identically to online,
> * and pins **any artifact type** via **version-aware, cloud-agnostic storage adapters** (e.g., S3 VersionId, GCS generations, Azure Blob versions, IPFS CIDs, NFS hashes).

---

## Git shortcomings for Agentic AI (and how ANVCS addresses them)

* **Conflict storms with many concurrent writers** → *CRDT live rooms* eliminate character‑level conflicts; **semantic merge** runs at snapshot time only.
* **History opacity & weak accountability** → **AI‑aware provenance** per commit (prompt, model, agent id, tests, env) + signed commits/notes.
* **Excessive commit churn/noise** → **Commit policy** (batch windows, formatting lanes) + **preview → confirm** flow to group intent, not keystrokes.
* **Fragile merge/rebase for automated edits** → **DAG streams** encode parallelism explicitly; no rebases required; merges are AST/test‑gated.
* **No contextual blame for AI work** → **Blame with provenance**: who/what/why (agent + prompt) rather than only “author line owner”.
* **Cross‑repo drift for multi‑service changes** → **Meta‑commit groups** pin coordinated commits across repos; CI enforces atomicity.
* **Non‑text & large artifacts are second‑class** → **Pluggable adapters** (S3/GCS/Azure/IPFS/NFS); exact pinning `{key, version_id|generation|cid, sha256}`; **lazy hydration**.
* **Limited modality diffs** (JSON, notebooks, IDLs) → **Modality‑aware merge drivers** (schema/cell/symbol diffs) by default.
* **No live visibility into agent actions** → **Live cursors & diff heatmap**, **activity timeline**, and **time‑lapse playback** of CRDT ops.
* **Offline/air‑gapped constraints** → **Disconnected mode parity** (same commits/metadata offline); **rejoin** sync later; local blobstores supported.
* **Networking realities (NAT/proxies/VPNs)** → **P2P WebRTC or relay** with end‑to‑end encryption; full **proxy** and **mTLS/JWT** support.
* **Weak policy/governance controls** → Path‑level **scopes** (read/write/commit/merge), rate limits, and **audit** trails for merges/snapshots.
* **Difficult debugging of rapid AI edits** → **Operation journal** + **time‑travel replay** to reproduce and bisect agent behavior.
* **Slow reproducibility across machines** → **Deterministic checkouts** with pinned artifacts and environment capture in commit metadata.

---

## Status & Scope

**Stability index**

| Area                                  | Status        | Notes                                                              |
| ------------------------------------- | ------------- | ------------------------------------------------------------------ |
| Specification (docs, schemas)         | **DRAFT**     | APIs and terms may change without deprecation.                     |
| Reference implementation (if present) | **INCUBATOR** | For exploration/testing only; not API-stable or security‑hardened. |

**Scope**

* Architecture and protocols (CRDT ops, DAG snapshots), AI provenance metadata, storage adapter interfaces.
* Cloud‑agnostic artifact versioning (S3 VersionId, GCS generations, Azure Blob versions, IPFS CIDs, NFS/content‑hash).

**Out of scope (for now)**

* A hardened, production implementation. Any implementation here is **experimental** and subject to rapid change.

If you are evaluating ANVCS for adoption, treat this repo as an **architecture reference**. Independent implementations are welcome to target the spec.

---

## Quick links (paths reflect current repo tree)

* 📘 **Core Spec v0.4 (main)** → [`docs/anvcs_core_spec_v_0.4.md`](docs/anvcs_core_spec_v_0.4.md)
  ↳ PDF: [`docs/anvcs_core_spec_v_0.4.pdf`](docs/anvcs_core_spec_v_0.4.pdf)
* 🧩 **Agent Streaming API (v0.2 addendum)** → [`docs/archive/05_anvcs_core_spec_addendum_v0.2_agent_streaming_api.md`](docs/archive/05_anvcs_core_spec_addendum_v0.2_agent_streaming_api.md)
* ☁️ **Codespaces & Remote Dev (v0.2 addendum)** → [`docs/archive/04_anvcs_core_spec_addendum_v0.2_codespaces_remote_dev.md`](docs/archive/04_anvcs_core_spec_addendum_v0.2_codespaces_remote_dev.md)
* 🤝 **Multi‑Agent Extensions & Live Observability (v0.3 addendum)** → [`docs/archive/06_anvcs_core_spec_addendum_v0.3_multi_agent_extensions_live_observability.md`](docs/archive/06_anvcs_core_spec_addendum_v0.3_multi_agent_extensions_live_observability.md)
* 🧠 **Shared CRDT Memory (v0.4 addendum — experimental)** → [`docs/archive/10_anvcs_core_spec_addendum_v0.4_crdt_shared_memory_space.pdf`](docs/archive/10_anvcs_core_spec_addendum_v0.4_crdt_shared_memory_space.pdf)
* 🗄️ **Docs index (archive)** → [`docs/archive/README.md`](docs/archive/README.md)
* 📚 **Docs index (root)** → [`docs/README.md`](docs/README.md)

---

## Why ANVCS?

Git is brilliant for human‑paced teams—but multi‑agent, 24/7 edits collide. ANVCS overlays **CRDT** live collaboration and a **semantic‑aware DAG** while keeping Git as the interoperable, immutable backbone.

## Core ideas

* **CRDT rooms** for live, conflict‑free edits (agents + humans).
* **Commit = semantic snapshot**: append‑only DAG node + mirrored Git commit with **AI provenance** (prompt, model, tests, artifacts).
* **Semantic merge** at commit boundaries (AST/test‑guided; optional LLM assistance).
* **Offline‑first parity**: identical UX/CLI and commit format on or off the network.
* **Any file type**: **pluggable, cloud‑agnostic adapters** (S3/GCS/Azure/IPFS/NFS) with version identifiers (VersionId/generation/content‑address).

## Architecture (high level)

```
Editors/Agents/CLI ──(WS/WebRTC/gRPC)──► CRDT Core ─► DAG Manager ─► Git Mirror
                                          │                           (.git/notes)
                                          └──► Semantic Merge ───► Storage Adapters
                                                           (S3/GCS/Azure/IPFS/NFS)
```

---

## Implementation tracks

If/when a reference implementation is added, it will be clearly separated:

```
.
├─ docs/                       # specification (authoritative)
├─ schemas/                    # JSON Schema, OpenAPI (machine‑readable spec bits)
├─ packages/                   # (optional) reference impl modules — INCUBATOR
│  ├─ anvcs-cli/               # CLI (experimental)
│  ├─ anvcs-relay/             # relay (optional) — experimental
│  └─ adapters/                # storage adapters (S3/GCS/Azure/IPFS) — experimental
├─ extensions/
│  └─ vscode/                  # VS Code extension — experimental
└─ examples/                   # small repos + fixtures for conformance
```

> **Important:** Any code here is for **experimentation** and **conformance fixtures**. Do not assume stability, performance, or security hardening.

---

## Getting started (spec first)

1. Read the **Core Spec**: [`docs/anvcs_core_spec_v_0.4.md`](docs/anvcs_core_spec_v_0.4.md).
2. Explore addenda for multi‑agent visibility, shared memory, and Codespaces/offline setups (see links above).
3. Integrate agents via the **Agent Streaming API** draft.

---

## Contributing

* Open issues/RFCs for gaps in the spec, terminology, and invariants.
* PRs welcome for **additional adapters**, **schemas**, **conformance tests**, and editorial fixes.
* If contributing experimental code, please gate it behind **INCUBATOR** labels and avoid breaking doc paths.

---

## Security & disclosure

This repo may include **prototype code**. Assume it is **not** security‑hardened. Please report vulnerabilities privately (see `SECURITY.md` when available).

---

## License

* **Docs/spec text:** CC BY‑4.0
* **Schemas/examples:** Apache‑2.0
