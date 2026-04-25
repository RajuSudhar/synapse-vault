# SynapseVault — System Design

**Status:** Canonical. Supersedes ADR-001, TECHSPEC-001, and FEASIBILITY-001 where they conflict.
**Date:** 2026-04-15
**Author:** Sudhar

---

## 0. Essence

SynapseVault is a **memory substrate for AI agents** — a typed, scoped, commit-aware store of claims an agent needs to be useful in a context that is not derivable from the source code itself. The substrate is a Rust core with a thin TypeScript SDK. Coding-agent memory and cross-repo flow-tracing are the first two modules built on it; any Claude SDK app can use the substrate directly as a "minified mem0."

The non-negotiable invariants:

1. **Every claim is pinned to a commit (or working-tree fingerprint) and an audience scope.** Memory is never a free-floating sticky note.
2. **Every read is parameterized by the requester's reality** (`HEAD`, dirty files, PR diff). The substrate projects memory onto the requester's world, not the world the memory was written in.
3. **Modules don't know about each other.** They share a substrate; they don't share types. Cross-module interaction is event-driven through the substrate.
4. **The SDK is the product.** MCP, CLI, agent, skill, Slack are reference adapters over the SDK.

---

## 1. Requirements

### 1.1 Functional

1. Store and retrieve typed memory entries (conventions, feedback, decisions, references, glossary).
2. Scope every entry as `Project`, `User`, or `Local { machine_id }` with workspace+identity routing on the remote.
3. Pin every entry to a commit SHA (or `working-tree` for in-progress claims) at write time.
4. Project memory onto a `RequestContext { head, dirtyFiles?, prDiff? }` at read time, partitioning results into stable / volatile / superseded / needs-review.
5. Run a configurable validator chain on writes (path exists, symbol exists, commit-pin present, audience matches scope).
6. Support a two-phase write lifecycle: `Pending` (cheap, agent-authored) → `Active` (validated, optionally human-approved).
7. Reconcile memory against new commits via webhook or scheduled job; auto-repin when citations still resolve, flag `NeedsReview` when they don't.
8. Record `supersedes` chains so memory has a queryable history of what we used to believe.
9. Provide BM25 + optional vector search across entries within a scope.
10. Expose all of the above through five adapter surfaces: SDK (TS), MCP server (stdio + Streamable HTTP), CLI, Claude Agent, Skill pack, Slack bot.

### 1.2 Non-functional

| Concern | Target |
|---|---|
| Read latency (assemble) | p95 < 50ms local, < 200ms remote |
| Write latency (pending) | p95 < 20ms local, < 100ms remote |
| Reconcile throughput | 10k entries / minute against a single commit diff |
| Substrate footprint | Rust core ≤ 5MB stripped; embeddable in any napi-rs or PyO3 host |
| Backend portability | Same trait runs over filesystem, SQLite+sqlite-vec, Postgres+pgvector |
| Privacy | Source code never required server-side; manifests + memory only |
| Auditability | Every entry has provenance (author, session, PR, commit) |
| Determinism | Same `(memory_state, ctx)` → same assembled output |

### 1.3 Constraints

- One-person project; pick boring, well-trodden crates and libraries.
- Must ship a usable v1 (local stdio MCP + SDK + memory module) before any remote-server work.
- Must be model-agnostic: no Anthropic-specific assumptions in core.

---

## 2. High-Level Architecture

```
                              ┌──────────────────────────────────────────┐
                              │            Adapter Surfaces               │
                              │  (each is a thin shell over the SDK)     │
                              ├──────────────────────────────────────────┤
   stdio / HTTP    ┌──────────┤  MCP server   CLI   Agent   Skill  Slack │
        ▲          │          └─────────────────┬────────────────────────┘
        │          │                            │
        │          │                  ┌─────────▼─────────┐
        │          │                  │ synapse-sdk (TS)  │   ← public API
        │          │                  │  napi-rs bindings │
        │          │                  └─────────┬─────────┘
        │          │                            │
        │          │     ┌──────────────────────┴───────────────────────┐
        │          │     │              Modules (Rust)                  │
        │          │     │  ┌─────────────────┐    ┌─────────────────┐  │
        │          │     │  │ synapse-coding  │    │  synapse-flow   │  │
        │          │     │  │ (T1-T4 tiers,   │    │  (manifests,    │  │
        │          │     │  │  glossary,      │    │   repo groups,  │  │
        │          │     │  │  conventions)   │    │   trace runs)   │  │
        │          │     │  └────────┬────────┘    └────────┬────────┘  │
        │          │     └───────────┼──────────────────────┼───────────┘
        │          │                 ▼                      ▼
        │          │     ┌──────────────────────────────────────────────┐
        │          │     │           synapse-substrate (Rust)           │
        │          │     │  Scope · Entry · RequestContext · Validator  │
        │          │     │  StorageBackend · LlmClient · VectorIndex    │
        │          │     │  Reconciler · DecayManager · EventBus        │
        │          │     └──────────────────────┬───────────────────────┘
        │          │                            │
        │          │   ┌────────────────────────┼────────────────────────┐
        │          │   ▼                        ▼                        ▼
        │          │ FileBackend         SqliteVecBackend         PgVectorBackend
        │          │ (~/.synapse,        (single-user laptop      (team remote,
        │          │  .ai/)              embedded)                 multi-tenant)
        │          │
        └──────────┘  Remote MCP server (Streamable HTTP) wraps PgVectorBackend
                      + identity layer (OIDC / API key) + reconcile workers.
```

**Substrate** is generic. It does not know what a "tier" is, what `glossary.md` means, or that flow tracing exists. It knows scopes, entries, partitions, embeddings, and decay rules.

**Modules** are layered consumers of the substrate. `synapse-coding` defines tiers (T1/T2/T3/T4), the canonical markdown format, and the convention validators. `synapse-flow` defines repo groups, manifests, and the picker→analyzer pipeline. They never depend on each other.

**Adapters** are shells that translate transport-level requests (MCP tool calls, CLI args, HTTP) into SDK calls. Adding a new surface = writing a new adapter; nothing in core or modules changes.

---

## 3. Core Types (substrate)

```rust
// ─── Identity & Scope ────────────────────────────────────────────────

pub enum Scope {
    Project { workspace_id: String },
    User    { user_id: String },
    Local   { machine_id: String, user_id: String },
}

pub struct WorkspaceCtx {
    pub workspace_id: String,
    pub repo_roots:   HashMap<RepoId, PathBuf>,  // workstation-resolved
    pub head:         CommitSha,
    pub dirty_files:  Vec<RelPath>,
    pub pr_diff:      Option<DiffSummary>,
    pub branch:       Option<String>,
}

// ─── The atomic unit ──────────────────────────────────────────────────

pub struct Entry {
    pub id:            EntryId,
    pub scope:         Scope,
    pub namespace:     String,         // module-defined ("coding/feedback", "flow/manifest")
    pub partition:     String,         // module-defined ("integration-tests", "breeze/frontend")
    pub body:          String,         // canonical markdown
    pub cited_paths:   Vec<RelPath>,
    pub cited_symbols: Vec<SymbolRef>,
    pub pinned_at:     Pin,            // CommitSha | WorkingTree { fingerprint }
    pub status:        Status,
    pub audience:      Audience,       // Self | Workspace
    pub supersedes:    Option<EntryId>,
    pub provenance:    Provenance,     // author, session, pr, tool
    pub created_at:    Timestamp,
    pub embedding:     Option<Vec<f32>>,
}

pub enum Status {
    Pending,
    Active,
    NeedsReview { reason: ReviewReason, since: CommitSha },
    Archived    { reason: ArchiveReason },
    Superseded  { by: EntryId },
}

// ─── Read-time projection ─────────────────────────────────────────────

pub struct AssembledMemory {
    pub stable:        Vec<Entry>,   // citations untouched by ctx
    pub volatile:      Vec<Entry>,   // citations overlap dirty/PR
    pub needs_review:  Vec<Entry>,   // flagged by prior reconcile
    pub superseded:    Vec<Entry>,   // included only if explicitly requested
    pub render:        String,       // formatted prompt-ready text
}

// ─── Write-time validation ────────────────────────────────────────────

pub trait Validator: Send + Sync {
    fn validate(&self, entry: &Entry, ctx: &WorkspaceCtx) -> Result<(), ValidationError>;
}

// Built-ins: PathExists, SymbolExists, CommitPin, AudienceMatchesScope,
//            NoSecretsLeaked, MaxBodySize.

// ─── Reconciliation ───────────────────────────────────────────────────

pub enum ReconcileAction {
    Keep,
    Repin(CommitSha),
    FlagNeedsReview(ReviewReason),
    Promote { from: EntryId },        // pending → active on PR merge
    Archive(ArchiveReason),
}

// ─── Backends (pluggable) ────────────────────────────────────────────

pub trait StorageBackend: Send + Sync {
    fn read   (&self, scope: Scope, ns: &str, part: &str) -> Result<Option<Vec<Entry>>>;
    fn write  (&self, entry: Entry) -> Result<EntryId>;
    fn search (&self, scope: Scope, ns: &str, q: &Query) -> Result<Vec<Entry>>;
    fn list_partitions(&self, scope: Scope, ns: &str) -> Result<Vec<String>>;
    fn delete (&self, id: EntryId) -> Result<()>;
}

pub trait VectorIndex: Send + Sync { /* upsert, query, delete */ }
pub trait LlmClient:   Send + Sync { /* complete, embed */ }
pub trait EventBus:    Send + Sync { /* publish, subscribe */ }
```

These six traits (`StorageBackend`, `VectorIndex`, `LlmClient`, `EventBus`, `Validator`, plus `ScopeResolver` for identity routing) are the entire pluggable surface. Everything else in the substrate is concrete.

---

## 4. Read & Write Flows

### 4.1 Write (the disciplined path)

```
agent → sdk.write(NewEntry)
     → module's NewEntry → substrate Entry transform
     → ValidatorChain.run(entry, ctx)              ← path/symbol/audience checks
     → status := Pending if agent-authored,
                 Active  if module trusts caller (e.g. CI promotion)
     → backend.write(entry)
     → eventbus.publish(EntryWritten { id, scope, ns })
     → return EntryId
```

`Pending` is cheap; `Active` is the promotion. Two ways an entry becomes `Active`:

1. **Manual promotion** via `sdk.commit(entry_id)` (re-runs validators).
2. **Auto-promote on PR merge** if `provenance.promote_on_merge = pr-N` and merge webhook fires.

### 4.2 Read (the projected path)

```
agent → sdk.assemble({ scope, query, ctx })
     → backend.search(scope, ns, query)            ← BM25, optionally vector-reranked
     → for each candidate:
           if ctx.dirty_files ∩ entry.cited_paths   → bucket: volatile
           if ctx.pr_diff touches entry.cited_*     → bucket: volatile
           if entry.status == NeedsReview            → bucket: needs_review
           if entry.status == Superseded             → drop (unless include_history)
           else                                      → bucket: stable
     → render markdown with section headers
     → return AssembledMemory
```

The agent's prompt sees explicit sections — *"Stable workspace memory"*, *"Memory possibly invalidated by your local changes"*, *"Memory flagged as needing review"*. The model is told which slice of memory is provisional, instead of being handed a flat list and forced to silently choose between memory and code.

### 4.3 Reconcile (the background path)

```
webhook (push to main) | scheduled (nightly)
  → diff := compute(old_head, new_head)
  → for entry in backend.list_active(workspace):
        action := reconciler.decide(entry, diff)
        apply(action)
  → eventbus.publish(ReconcileFinished { stats })
```

This is where `synapse-flow` listens for `EntryWritten { ns: "coding/decisions" }` and re-enriches its manifest with the new architectural fact, and where `synapse-coding`'s decay manager listens for `ReconcileFinished` to compact the warm tier. Cross-module behavior emerges from event subscriptions, not direct calls.

---

## 5. Storage & Deployment Topologies

### 5.1 Local solo (default for v1)

```
Editor (Claude Code)
  ↕ stdio
synapse-mcp-server (binary)
  ↕ napi
synapse-sdk (TS)
  ↕ ffi
synapse-core + synapse-coding + synapse-flow
  ↕
FileBackend → .ai/ (project, git-tracked)
              ~/.synapse/users/{uid}/ (personal)
              ~/.synapse/local/{mid}/ (workstation-only)
SqliteVecBackend (optional, for embeddings) → ~/.synapse/index.db
```

No server, no daemon, no auth. The MCP binary is forked per editor session.

### 5.2 Remote shared (team)

```
Developer machines    CI runners     Slack bot
       │                  │             │
       └─────────┬────────┴─────────────┘
                 │  HTTPS (Streamable HTTP MCP)
                 │  + OIDC bearer token
                 ▼
       synapse-mcp-server (long-lived service)
                 │
                 ├── PgVectorBackend → Postgres
                 ├── ReconcileWorker (consumes git webhooks)
                 └── PromotionWorker (consumes PR-merge webhooks)
```

What the remote server stores: **memory entries + manifests + embeddings**. What it does **not** store: **source code**. Manifests are built where the code lives (developer's machine or CI), then uploaded. The picker LLM call happens server-side using only manifests; the analyzer phase runs client-side where the working tree is.

This keeps the server free of VCS credentials, source code at rest, and per-repo build environments — and gives developers a real privacy story.

### 5.3 Hybrid (the realistic case)

A power user runs both: local stdio MCP for `Scope::Local` + workstation-specific tools, plus the remote MCP for `Scope::Project` and `Scope::User` data. The SDK transparently routes by scope. The remote MCP refuses to register tools whose handlers touch `Scope::Local` data — type-level enforcement of "this state isn't portable."

---

## 6. Modules

### 6.1 `synapse-coding` (memory module)

Defines the canonical 4-tier convention layered over the substrate:

| Tier | Substrate mapping | Purpose |
|---|---|---|
| T1 (hot) | `ns: "coding/hot"`, `partition: "core"`, ≤150 lines | Always loaded |
| T2 (warm) | `ns: "coding/warm"`, `partition: <domain>` | On-demand by domain |
| T3 (cold) | `ns: "coding/cold"`, `partition: <topic>` | Archive/history |
| T4 (graph) | `ns: "coding/graph"`, `partition: "structural"` | Glossary, dep map |

Adds coding-specific validators (`MarkdownStructure`, `CitationFormat`) and the sync engine that mirrors `.ai/` to `CLAUDE.md`, `.cursorrules`, etc.

### 6.2 `synapse-flow` (flow-tracer module)

Defines repo groups, manifests, and the picker→analyzer pipeline.

- Manifests: `ns: "flow/manifest"`, `partition: "{group}/{repo_id}"`. Path-relativized; resolved against `WorkspaceCtx.repo_roots` at use time.
- Repo groups: `ns: "flow/group"`, `partition: "{group}"`.
- Trace runs: `ns: "flow/trace"`, `partition: "{group}/{trace_id}"`.

Capacity enforcement (intent doc §6: "fail at registration, not at trace") lives here, not in the substrate. The picker (Sonnet-tier `LlmClient`) and analyzer (Opus-tier `LlmClient`) are configured per-module.

Cross-module hook: subscribes to `EntryWritten { ns: "coding/*" }` and re-enriches the corresponding manifest entry. Subscribes to `ReconcileFinished` to mark stale manifest entries for rebuild.

---

## 7. API Surfaces

### 7.1 SDK (the public API)

```ts
import { Synapse } from '@synapsevault/sdk';

const mem = new Synapse({
  backend: { kind: 'file', root: './.ai' },
  modules: ['coding'],          // opt-in modules
});

// minimal mem0-style usage
await mem.write({ scope: { user: 'alice' }, body: 'prefers dark mode' });
const ctx = await mem.assemble({ scope: { user: 'alice' }, query: 'theme' });

// full coding-agent usage
await mem.coding.writeFeedback({
  body: 'integration tests must hit a real DB',
  citedPaths: ['tests/integration/'],
  why: 'mock divergence broke prod migration last quarter',
});
```

### 7.2 MCP tools (reference adapter)

Memory: `memory_assemble`, `memory_write`, `memory_commit`, `memory_search`, `memory_supersede`, `memory_archive`.
Flow: `flow_register_group`, `flow_list_groups`, `flow_remove_group`, `flow_trace`, `flow_follow_up`, `flow_inspect_group`.
Workspace (remote only): `workspace_bootstrap`, `workspace_register_repo`.

Local-only tools (anything touching `Scope::Local`) are conditionally registered: the remote shell skips them.

### 7.3 CLI

`synapse init`, `synapse read`, `synapse write`, `synapse search`, `synapse reconcile`, `synapse sync`, `synapse flow trace`. Thin wrapper over the SDK; useful for git hooks and shell scripts.

---

## 8. Commit-Pinning & Reconciliation Discipline

This is the discipline that makes the system not lie:

1. **Every write captures `pinned_at`.** No exceptions. If the agent can't determine a commit, it pins to `WorkingTree { fingerprint: hash(dirty_files) }` and `audience := Self`.
2. **Every read carries `WorkspaceCtx`.** The SDK's default helpers compute `ctx` from `git rev-parse HEAD` + `git status`; the MCP shell does the same per-request.
3. **Reconcile is mandatory on commit.** Webhook in remote, `post-commit` git hook in local. Without this, memory drifts silently.
4. **Supersede, don't overwrite.** Contradicting an existing entry creates a new entry with `supersedes: <old_id>`. History is queryable.
5. **`Audience::Self` is the firewall.** Personal exploration never reaches `Scope::Project`. The validator chain enforces this at write time.

---

## 9. Trade-offs Made Explicit

| Decision | Chosen | Rejected | Why |
|---|---|---|---|
| Core language | Rust | Go, TS-only | Single binary, FFI-friendly, tantivy native |
| Public API | TS SDK over napi-rs | Pure CLI, REST-only | Devs already in TS for Claude SDK apps |
| Default backend | Filesystem markdown | SQLite | Inspectable, diff-able, no server |
| Local vector | sqlite-vec | LanceDB, Chroma | Embedded, no daemon, active maintenance |
| Remote vector | pgvector | Pinecone, Weaviate | Self-hostable, single dependency |
| MCP transport | stdio + Streamable HTTP | SSE, custom | SSE deprecated 2025-03; HTTP is the path |
| Module isolation | Event bus, no direct deps | Shared types | Future modules must not couple |
| Flow source-of-truth | Manifests, not source clones | Server-side clones | Privacy, no VCS auth, no build infra |
| Scopes | Project / User / Local | Single global | Required by flow-tracer and personal mem |
| Write lifecycle | Pending → Active | Direct active | Cheap agent writes, validated promotions |

---

## 10. What Ships in v1

**In:**
- `synapse-substrate` crate (traits, types, validators, reconciler)
- `synapse-core` (substrate impl + FileBackend + tantivy BM25)
- `synapse-coding` module (T1/T2 tiers, sync engine)
- `synapse-sdk` (TS, via napi-rs)
- `synapse-mcp-server` (stdio only)
- `synapse-cli`
- Single-tenant local deployment

**Deferred (v2+):**
- `synapse-flow` module (full implementation)
- Remote Streamable HTTP server + auth + multi-tenant
- PgVectorBackend
- SqliteVecBackend (optional embeddings)
- Slack adapter
- Skill pack adapter
- Promotion-on-PR-merge workflow
- Workspace bootstrap tools

The v1 cut is deliberately the minimum that proves the substrate's shape. Everything in v2 is a backend/adapter swap, not a redesign — that's the test of whether the substrate boundaries are right.

---

## 11. Build Sequence

1. **Substrate spike** — define all traits and types in `synapse-substrate`, no implementations. Fits in a single PR. Compile-only.
2. **FileBackend + Validators** — concrete impl, write/read round-trip, validator chain working against a real `.ai/` directory.
3. **Reconciler** — exercise it with a contrived diff and assert the four `ReconcileAction` outcomes.
4. **synapse-coding T1/T2** — first module. Proves the substrate doesn't leak module concepts.
5. **napi-rs binding + SDK** — expose `read`, `write`, `assemble`, `commit`. End-to-end TS test.
6. **MCP stdio shell** — register six tools. Wire to Claude Code, dogfood on this very repo.
7. **CLI** — `synapse init/read/write/search/reconcile`.
8. **Decay + supersede chains** — the long-tail integrity work.

Each step is a merge-able milestone. Steps 1-4 are Rust-only; 5+ touches TS.

---

## 12. What I'd Revisit as the System Grows

- **Scope explosion.** If teams want `Team`, `Department`, `Project-within-workspace` distinctions, the `Scope` enum becomes a tree. Plan: keep `Scope` open for extension via `Scope::Custom { kind, key }`.
- **Embedding model lock-in.** `@xenova/transformers` for local is fine for v1; production teams will want their own model. Plan: `LlmClient::embed` is already an abstraction; swap is a config change.
- **Reconcile cost on huge monorepos.** Walking every entry on every commit doesn't scale past ~100k entries. Plan: secondary index `(cited_path → entry_id)` once it matters; not before.
- **Cross-workspace knowledge sharing.** A consultant working across five client workspaces will want to share their `User` memory across all of them. Plan: identity layer already supports this; UI question, not architecture.
- **Conflict resolution on concurrent writes.** Two agents writing contradictory memories simultaneously. Plan: last-write-wins with `supersedes` chain for v1; CRDT or operational transform if it bites.

---

## Appendix: Glossary delta from prior docs

- "T1/T2/T3/T4" is now a `synapse-coding`-module concept, not a substrate concept. Substrate sees only `(scope, namespace, partition)`.
- "Manifest" is path-relativized; absolute paths are resolved at the workstation via `WorkspaceCtx.repo_roots`.
- "Local-only flow tracer" from FEASIBILITY-001 is **superseded**: flow tracer is *transport-portable* but `Scope::Local` data within it (workstation roots) stays local.
- "memory_propose / memory_commit" replaces "memory_write" as the canonical write surface; `memory_write` is shorthand for `propose + commit`.
