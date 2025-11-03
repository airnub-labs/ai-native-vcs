# ANVCS Addendum v0.4 — Shared CRDT Memory (Repo‑Scoped Agent Context)

**Status:** Draft • **Scope:** Specification Addendum • **Targets:** ANVCS Core v0.4 and later

---

## 1) Motivation

Agentic AI teams suffer from fragmented, short‑lived context: one agent’s discoveries are buried in logs, external vector stores, or ephemeral scratchpads. Developers lack a single, auditable place to see what the agents “know” right now. This addendum proposes a **shared, CRDT‑backed memory space** that is:

- **Repo‑scoped and local‑first** (works offline, syncs later),
- **Git‑compatible and immutable** at snapshot points (append‑only DAG),
- **Human‑inspectable** (JSON/Markdown), and
- **Agent‑friendly** (low‑latency, conflict‑free live updates).

Agents and humans collaborate in a **Memory Room** parallel to code rooms. When memory reaches **semantic significance** (goal achieved, design decided, invariant established), it is **snapshotted** into a Git commit side‑by‑side with code, preserving provenance.

---

## 2) Concept

**Memory Room** = a CRDT document (or set of documents) mounted under `memory/` in the repo.

- Live updates are CRDT operations that **always converge** (no conflicts).
- Snapshots produce a **DAG node + Git commit**; memory files are part of the tree.
- The latest memory is the working set; **history** remains in Git for audits and time‑travel.

### Scope levels
- **Per‑repo (baseline)**: `memory/` folder tracked in the repo (default).
- **Org/global (optional future)**: separate “meta‑memory” repos, referenced via **meta‑commit groups**, to share cross‑repo knowledge.

---

## 3) Architecture Overview

```
Editors/Agents/CLI ──(WS/WebRTC/gRPC)──► CRDT Core ─► Memory Manager ─► Git Mirror
                                           │            (memory/)        (.git/notes)
                                           └──► Semantic Snapshot ───► DAG Commit
```

**Key pieces**
- **CRDT Core:** operation journal (Yjs/Automerge class), offline journals, GC.
- **Memory Manager:** schema validation, lanes/namespaces, policy & ACL, indexing hooks.
- **Git Mirror:** dual‑write commit; AI metadata in notes or `.ai/meta/`.
- **Semantic Snapshot:** explicit action when memory gains semantic meaning; optional tests/guards.

---

## 4) Data Model

### 4.1 Memory entry (structured)
```json
{
  "id": "mem_01HZX...",
  "type": "fact|plan|decision|link|note",
  "scope": "repo",              
  "topic": "build:semantic-merge",
  "tags": ["merge","ast","design"],
  "content": {
    "text": "AST-based merge should gate overlapping edits in /src/**/*.ts",
    "links": ["code://src/merge/engine.ts#L120"],
    "constraints": ["run unit suite before merge"]
  },
  "provenance": {
    "agent_id": "merge-bot",
    "prompt_ref": ".ai/meta/commits/abc123/prompt.txt",
    "model": "gpt-4o-2025-10"
  },
  "importance": 0.87,
  "retention": { "ttl_days": 180, "archive_on_expiry": true },
  "created_at": "2025-11-03T00:00:00Z",
  "updated_at": "2025-11-03T00:02:15Z"
}
```

### 4.2 Memory files
- **Structured:** `memory/state.jsonl` (append‑only logical entries), `memory/index.json` (optional summary), `memory/lanes/*.jsonl`.
- **Unstructured:** `memory/notes/*.md` with heading‑level anchors.

**CRDT mapping**
- Maps/Sets for keyed entries (stable IDs).
- Ordered lists for plans/queues.
- Rich‑text CRDT for `.md` notes.

---

## 5) CRDT Design Options

| Content | CRDT form | Pros | Cons |
|---|---|---|---|
| Key→Value facts | Map CRDT | Deterministic merge, small ops | Requires ID discipline |
| Ordered lists (plans) | List CRDT | Natural task queues | Index maintenance |
| Notes/docs | Rich‑text CRDT | Human‑friendly | Heavier payloads |
| Tag sets | Set CRDT | Simple membership | None |

**Compaction:** periodic compaction merges tombstones; long‑term archival moves aged entries to `memory/archive/` while preserving Git history.

---

## 6) Security & Access Control

- **Scopes:** `memory:read`, `memory:write`, `memory:snapshot`, `memory:admin`.
- **Lanes/namespaces:** `memory/public/`, `memory/private/agent-X/`, `memory/team/<name>/`.
- **Encryption (optional lane):** age/NaCl public‑key encryption for sensitive lanes; cleartext only in working copy if authorized.
- **Commit signing:** memory snapshots signed like code commits.
- **Audit:** provenance embedded in commit metadata and per‑entry `provenance` fields.

---

## 7) CLI & API

### 7.1 CLI (additions)
```bash
ai-vcs memory ls [--lane team/search] [--query "tags:merge topic:semantic-merge"]
ai-vcs memory get <id>
ai-vcs memory write --type fact --topic semantic-merge \
  --content "AST-based merge should gate overlapping edits in /src/**/*.ts" \
  --tags merge ast --importance 0.8
ai-vcs memory note open notes/merge_strategy.md   # opens rich-text CRDT note
ai-vcs memory snapshot -m "doc: record merge policy v1" --lane team/merge
ai-vcs memory graph --since 7d --filter type:decision
ai-vcs memory export --format jsonl --out .ai/exports/memory.jsonl
ai-vcs rejoin-room   # also publishes buffered memory ops
```

### 7.2 API (REST/WS excerpts)
```http
POST /api/memory/ops           # batch CRDT ops
POST /api/memory/snapshot      # create DAG+Git snapshot
GET  /api/memory/query?q=...   # simple query (topic/tags/type)
WS  /ws/memory                 # live ops stream (op|ack|presence|error)
```

**Op envelope**
```json
{ "type": "op", "lane": "team/merge", "seq": 512, "agent": "merge-bot", "data": { /* CRDT op */ } }
```

---

## 8) IDE/UX

- **Memory Panel:** filter by type/topic/tags; highlight recent changes; per‑entry provenance.
- **Pin to Prompt:** select memory entries to inject into the agent’s system/context prompt.
- **Time‑lapse:** playback memory evolution; diff between snapshots.
- **Cross‑linking:** click `code://` links to open files/lines; show back‑refs from code to memory decisions.

---

## 9) Storage, Indexing & GC

- **Working set:** small, hot memory (current truths) kept under `memory/`.
- **Archive:** aged/low‑importance entries moved to `memory/archive/` by policy; still reachable via Git history.
- **Index overlay (optional):** build BM25/embedding indexes on snapshot; store under `.ai/memory/index/` for offline retrieval. (Indexes are caches; source of truth is `memory/`.)
- **Retention policies:** TTL on entries; compaction window; lane‑specific limits.

---

## 10) Offline & Reconnect Flows

- **Offline:** ops append to local CRDT journal; `ai-vcs memory snapshot` creates local Git commit exactly like online.
- **Reconnect:** publish ops; CRDT converges; memory snapshots are ordinary commits in DAG; no special handling required.

---

## 11) Adoption & Migration

- **Start small:** `memory/notes/*.md` for human‑first notes; add structured `state.jsonl` when ready.
- **Bridge vector stores:** importer to pull high‑value facts into `memory/` with provenance.
- **Policy first:** define lanes, ACLs, and snapshot criteria (what counts as “semantic significance”).

---

## 12) Open Questions

- **Bounded size:** how to cap working set without losing salient context? (importance/recency heuristics).
- **Prompt discipline:** who decides which memory entries feed into prompts? (agent role vs human override).
- **Confidentiality:** encrypted lanes vs shared discovery; redaction policies in logs.

---

## 13) Milestones

1. **MVP (local‑only):** CRDT memory (Map/List/Notes), CLI write/ls/get, snapshot to Git.
2. **Live sync:** WS/WebRTC ops; VS Code Memory Panel; time‑lapse viewer.
3. **Policies & lanes:** ACLs, encrypted lanes, TTL/GC; indexing overlay.
4. **Org/global:** meta‑memory repo + meta‑commit groups; cross‑repo search.

---

## 14) Compatibility

- **Works with ANVCS Core v0.4** (dual‑write to `.git/` + DAG; offline parity).
- **Codespaces/Proxies:** reuses ANVCS transports (WS/WebRTC/gRPC), proxy envs, and auth (mTLS/JWT/OIDC).
- **Air‑gapped:** local‑first by design; indexes optional; artifacts (if any) use same adapters.

---

**Summary:** This addendum defines a **repo‑scoped shared memory** that is CRDT‑synchronized for live collaboration and **snapshotted** into immutable Git commits when it matters. Agents and humans gain a single, auditable context that persists across sessions, scales offline, and integrates cleanly with existing ANVCS workflows.