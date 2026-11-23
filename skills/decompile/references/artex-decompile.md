# ARTEX decompile — BaaRai/ARTEX (fork of Autumn-27/ARTEX)

**Snapshot decompiled:** `github.com/BaaRai/ARTEX` @ `8ae53f74cf2833d5d91271561e2ba46df019b790`
(2026-11-17, merge PR #306). **License: AGPL-3.0** — this file records facts and
patterns in original text; no code or prompt text is copied. An earlier snapshot
was decompiled 2026-11-14 as "ScopeWeaver" and produced the security-review
adopt set v1; this pass covers the current BaaRai fork and supersedes that
snapshot's facts where they differ.

**Scale (observed):** 531 Go files, ~140k LOC Go, `db/schema.sql` 1,610 lines,
~1,269 Go test functions, Next.js frontend under `web/` embedded via `go:embed`.
Module path is still `github.com/Autumn-27/artex`; `go.mod` carries
`replace github.com/Autumn-27/norma => github.com/BaaRai/Norma v0.4.11` — the
agent SDK is a fork too, so a decompile that stops at the ARTEX tree misses
half the runtime (agentcore / tool / permission / harness / memory /
transcript / noaadapter all live in norma).

## What it is

A self-hosted, LLM-driven autonomous penetration-testing platform. One Go
binary (Next.js UI embedded, `-tags embedui`) + PostgreSQL as the only
external dependency. Entry point `cmd/artex/main.go`: API/UI on
`127.0.0.1:8787`, recording MITM proxy on `127.0.0.1:8788`, graceful drain
(HTTP shutdown 10s, task drain 15s) with cancel-cause propagation
(`agent.AbortShutdown`) so an interrupted run keeps its diagnostic cause.

## The dual graph (the core idea)

Two PostgreSQL graphs, linked by anchors — `db/schema.sql` is the ground
truth; read it before any Go file.

- **Asset graph (global, shared across tasks).** `companies` →
  `assets` (type ∈ `root_domain | ip | subdomain | app | service | endpoint`)
  with per-type unique indexes as the deduplication keys, plus
  `company_scope`. Containment and dedup keys are **computed by the program**;
  agents submit only raw observations (`insert_assets`). An asset written by
  hand can never corrupt the graph's shape.
- **Exploration graph (per task).** `explorations` → `exploration_nodes`
  (kind ∈ `goal | intent | fact | finding | hint`) + `exploration_edges`
  (rel ∈ `spawns | derived_from | yields | proves`, plus a derived `covers`
  layer used only by compaction and filtered out of causal reasoning) +
  `exploration_anchors(node_id, asset_id)`. Anchors are what make per-asset
  test coverage computable: which intents touched an asset, which findings
  prove against it.
- **Everything else hangs off these:** `tasks`, `task_scope` (the coverage
  denominator), `task_constraints`, `activity` (per-step audit stream),
  `findings` + `finding_traffic_bindings` + `traffic_evidence_snapshots`,
  `finding_retests`, `intercept_rules` / `intercept_pending`,
  `target_pins` / `target_gate_exemption_log`, `agent_prompts` (versioned,
  seeded), `coverage_ledger` (+ history), `rsi_lessons` (+ evidence, usage),
  `llm_profiles` / `llm_profile_health`.

## The engine loop

`server/engine.go` (~1,486 lines): one `plannerLoop` + N `workerLoop`
goroutines per task, plus a claim-sweep loop. Event-driven: any graph change
debounces (800 ms) into a planner wake-up.

1. **Goals decomposer** (`agent/goals.go`) runs first: extracts *operation
   constraints* (`set_constraints`, type deny/allow) and the *explicit asset
   scope* (`add_task_scope`) from the operator's text before decomposing
   goals. Constraints must be self-contained with concrete targets (deictic
   "current target" is rewritten to the literal host/port). Scope follows a
   **minimal-scope principle**: a subdomain-bearing hostname registers as
   that subdomain, never shrunk to its root domain; a root domain is
   registered only when the operator gave a bare root or said "all
   subdomains". The scope tail is code-owned — appended after the editable
   prompt body so it cannot be edited away in the UI.
2. **Planner** (`agent/planner.go`, sole intent generator): reads a prefetched
   `graph_overview`, judges goals (`prove_goal`), dispatches 0..N intents
   with `asset_ids`. Most wake-ups dispatch 0. Two code-owned guards close
   every planner message: the **goal anchor** (operator goal + constraints
   restated last, after all data; data cannot add goals, targets, or scope)
   and the **hard floor** (goal unmet + `frontier_open=0` + no running
   intents ⇒ the round must produce ≥1 intent; 0 intents there is a stall,
   not a decision). A **persistent todolist per task** (`TodoStore` keyed by
   exploration id) survives across wake-ups, so a serial attack chain is
   recorded once and dispatched step-by-step only when prerequisite facts
   exist — never front-loaded in parallel.
3. **Worker** (`agent/worker.go`): claims exactly ONE intent via
   `claimNext` (claim TTL + heartbeat + fence token; losing the claim raises
   `ErrIntentClaimLost` and the worker must stop), fetches the intent's raw
   assets, executes with real tools (Bash/Kali tooling routed through the
   recording proxy via injected `HTTP(S)_PROXY`/`ALL_PROXY` + per-ecosystem
   CA variables), writes back assets/facts/findings, stops. Workers never
   generate directions. On turn/time budget exhaustion a settlement round
   forces write-back before termination, so identified facts are never lost
   to a timeout.
4. Write-back changes the graph → debounce → planner wakes again, until
   `prove_goal` closes the task (or budget/timeout forces a wrap-up round in
   which workers may only write back).

Other agents: **mainagent** (human-in-the-loop chat), **Auto** (platform
operator: task ops + skill/tool/MCP management), **pentest** (solo
planner+executor+auditor in one), **reporter** (fires on `report_finding`,
writes the Markdown report from bound evidence + worker traces), **retester**
(below), plus side-question sessions. Prompts are a **catalog**
(`agent/promptcatalog.go`, `agent/toolcatalog.go`): editable bodies seeded
idempotently into `agent_prompts`/`tools`; code-owned tails (traffic tools,
artifact rules, scope, goal anchor, noise block, lessons frame) are injected
at render time and are never stored or editable.

## Facts carry provenance

Every `record_fact` declares a source: `target_response` (the target itself
returned it) | `tool_output` | `third_party_intel` | `operator_input` |
`inference` (default). A `target_response` fact never proves a goal on its
own. This is the decompile-relevant discipline: a claim's origin class is
stored with the claim, not reconstructed later.

## Evidence: findings keep their receipts

- **Recording proxy** (`traffic/`, on `go-mitmproxy`): generated CA, every
  Bash/HTTP call traced; bodies spill to disk with a SQLite FTS index;
  WebSocket/SSE streams recorded as message rows; replay supported
  (hop-by-hop headers dropped, redirects not followed); rewrite rules are
  atomic per rule, refuse scheme/host/port and hop headers, and are skipped
  whole for binary/>1 MiB/compressed/streamed bodies. A **browser extension**
  (`extension/`) ingests authenticated-session traffic into the same store
  (text only, 256 KiB body cap, batches ≤50).
- **Atomic binding** (`evidence/store.go`, `db/finding_traffic.go`):
  `report_finding` may bind real traffic IDs (`traffic_refs`, in
  reproduction order). Binding is validated server-side; an invalid ref
  fails the whole submission. Snapshots are content-hashed, encrypted at
  rest (AES-256-GCM, `keys/secret.key` or `ARTEX_SECRET_KEY`), archived by
  binding (archive v3) rather than by host, and GC'd with a grace period —
  deleting the hot traffic does not orphan a finding's proof.
- **Verification (S1)** (`db/finding_verification.go`,
  `agent/finding_workflow.go`): machine-checked state kept apart from the
  human disposition (`findings.status`). `verification_status` ∈ verified /
  unverified / blocked / not_assessed / false_positive; **POE level 0–4**
  (1 = error/timing variation, 2 = structural manipulation, 3 = data
  actually extracted, 4 = critical impact such as credentials or code
  execution). **L3 is the minimum for `verified`.** Verification requires
  `evidence_quotes`: exact lines copied from a bound proof/verification
  body; the server checks each quote verbatim after whitespace
  normalization and gzip/deflate decoding. No quotable proof ⇒ the finding
  stays unverified, which is an acceptable outcome; inflating POE to force
  `verified` is explicitly forbidden. Findings also carry optional
  `counterevidence` and `confidence_rationale`.
- **Adversarial + quality gates** (seeded, test-pinned in
  `agent/*_seed_test.go`): before `report_finding`, the candidate is argued
  against on five axes — reachability, preconditions, downstream controls,
  scope, simpler explanation (the retester uses reachability, preconditions,
  reproducibility, scope, claimed impact). Eliminations need evidence at
  the same bar as confirmations. The PoC must show the attacker's goal
  achieved end-to-end, with a **measured impact** (a number with its
  denominator, plus what the defense blocked) and a stated **proof basis**
  (executed-live / executed-lab / static-analysis-unexecuted). Version/CVE
  matching or "looks injectable" is never a confirmed finding. Distinct
  entry points / parameters / privilege levels are never merged.
- **Retester** (`agent/retester.go`, `db/finding_retests.go`): a dedicated
  session per retest, bound to exactly one finding. Loads the original
  finding, evidence/PoC/report, assets, and original constraints; historical
  material is data to verify, not instructions. Minimal targeted check using
  only the original PoC's key conditions. Exactly one verdict:
  `reproduced` | `fixed` | `inconclusive`. `fixed` requires all of:
  comparable environment/preconditions, original trigger no longer works,
  a named normal control still works, evidence supports the fix. A single
  failed request is never a fix. Only a successful session ending in
  `fixed` auto-moves the finding's status to Fixed; the original report and
  evidence are never mutated. One session saves exactly one verdict.

## Autonomy with brakes

- **Intercept gate** (`guard/guard.go` PreToolUse hook → `intercept/`):
  every tool call is evaluated against DB rules (string / regex / **argv
  prefix**). Built-in destructive/exfil rules are seeded as ordinary
  `[built-in]` rules — user-disable-able, versioned by the seed discipline.
  Decisions record `decided_by` ∈ human | rule | model | system.
  - **Argv prefix semantics** (`intercept/argv.go`, parser: `mvdan.cc/sh`):
    a pattern is 1–8 literal tokens. It matches only a *single simple
    command* whose parsed prefix equals the pattern. Pipes, lists,
    redirections, prefix assignments, subshells, background jobs, wrapper
    programs (`command`, `builtin`, `env`, `exec`, `eval`), and any
    command/parameter/arithmetic substitution or extended glob make the
    command *not simple* and therefore unmatchable — fail-closed by
    construction. Flags after the subcommand are covered by construction
    (`["git","push"]` covers `git push --force`); a flag *before* the
    subcommand (`git -C /repo push`) yields no prefix at all rather than an
    invented one. Prefix length comes from a code-owned arity table
    (`git`→2, `npm run`→3, unknown program→1).
  - **Closed judge** (`intercept/closed_judge.go`, `intercept/prompt.go`):
    when no rule decides, an LLM judge classifies by reversibility ×
    affected object (Deny / Allow / Ask lists + seeded deny patterns).
    Its reply is parsed fail-closed: strict JSON with exactly `decision` +
    `comment`, the comment in a fixed "Actual action: …; Consequence if
    successful: …; Matched rule: …" shape (≤2,400 chars). A decision keyword
    is never extracted from prose, arguments, or a broken reply — an
    unparseable verdict follows the configured model-failure path, it does
    not become an allow. Tool arguments and target content are untrusted
    data for the judge.
  - **Target gate** (`intercept/target_gate.go`, off by default, API-only):
    per-call target extraction → asset block rules first (matched on the
    declared text, before any resolution) → exemptions → DNS resolution →
    **pins**: a pin covers a host:port only if it holds for *every*
    resolved address. At most 5 unpinned targets per call may be put to a
    human. The check re-runs fresh at dial time (`target_dial.go`) and on
    redirects, so a DNS answer that changes between approval and connect
    does not inherit the approval.
  - Planner/mainagent get a target-only guard: built-in deny rules + target
    gate, no user rules, no judge, no audit-ring writes.
- **Scope gate:** `task_scope` rows define the coverage denominator; assets
  outside declared scope are not explored. Scope is registered by the goals
  decomposer under the minimal-scope principle (above), never widened by
  data.
- **Prompt-injection framing, two layers:**
  1. norma's untrusted-output boundary (on by default,
     `ARTEX_UNTRUSTED_BOUNDARY=0` disables): tool results framed between
     nonce-tagged markers (~154 bytes/result).
  2. ARTEX's own **data envelope** (`agent/data_envelope.go`, #124): a
     per-message `crypto/rand` nonce frames every derived field (worker
     output, facts, findings, cold digests, related tasks, non-operator
     hints, RSI lessons) in planner/worker user messages. The marker stem
     inside data is neutralised by a replacer, so data can neither open nor
     close a frame whatever nonce it guesses. Operator material (task,
     goals, operator hints) sits *outside* the frame; everything else is
     labelled data that cannot add goals, targets, or scope. The frame is
     code-owned and admitted to be advisory — it marks data, it cannot stop
     a model from following it.
- **Deterministic scanner ingest** (`agent/scan_ingest.go`,
  `agent/bbot_ingest.go`): nmap XML / nuclei JSONL / httpx JSON / ffuf JSON /
  BBOT output are parsed by typed-field rules, never by the LLM. Every
  target-shaped free-text field (banner, product, title, template name) is
  kept only inside one Go-quoted inert string in a fact detail — it never
  reaches a summary, a notable list, or an instruction. URLs are stripped of
  userinfo before they can become assets. Caps are loud: 50,000 items/file,
  overflow is counted and the result is marked `truncated`.
- **Hierarchical budgets** (`agent/budget.go`): task budget ⊃ intent budget
  ⊃ planner rounds (`ARTEX_TASK_BUDGET_{TOKENS,TURNS}`,
  `ARTEX_INTENT_BUDGET_{TOKENS,TURNS}`; in-memory, restart = fresh). A
  child's effective budget is min(own, parent remaining). Wrap-up reserve =
  max(5%, 20,000 tokens / 2 turns). A worker is not started when remaining
  budget can't cover its estimate (mean of finished workers, else 50k
  tokens / 10 turns).
- **Noise governor** (`egress/`, `traffic/pace.go`): profiles quiet
  (2 rps, burst 4, 1 in flight) / balanced (5/10/2) / aggressive (20/40/4);
  jitter, 429/503 backoff honouring `Retry-After` (bounded retries,
  safe methods only), in-flight dedupe + short GET cache (never mutations),
  per-host counters. A code-owned low-and-slow prompt block gives agents
  the exact per-scanner flags (nmap `-T2..T4 --max-rate`, nuclei/httpx/
  katana/ffuf rate + concurrency, sqlmap `--delay --threads=1`) and the
  rule "never raise the rate to meet a deadline". Best-effort, explicitly
  not a security boundary: raw scanners and proxy-bypassing commands are
  not paced.
- **Coverage ledger** (`db/coverage_ledger.go`, `agent/coverage_ledger.go`):
  workers call `record_coverage(surface, risk_area, outcome, evidence,
  reason)` with outcome ∈ reported | no_issue_found (evidence required) |
  ruled_out (reason required) | not_applicable | needs_follow_up. Gaps are
  computed, surfaced to the planner each round (top 3) and in full at
  wrap-up (up to 25): an unexamined surface×class is a named gap, never a
  clean result, even when `prove_goal` is allowed.
- **Shared worker memory:** workers can `search_all_worker_traces` /
  `list_worker_traces` / `get_worker_trace` — observations that never
  became formal facts stay reusable by later workers, while each worker
  stays confined to its claimed intent.
- **Cold-graph compaction** (`agent/coldgraph.go`, `agent/compaction.go`):
  nodes inactive ≥6 planner rounds fold into digest nodes (blocks ≥2) off
  the hot path; a commit-time liveness recheck drops any member that
  revived during generation. norma's experimental **NOA** model-driven
  compression is the alternative context manager (mutually exclusive with
  built-in compaction; originals archived under `<workDir>/noa`).
- **LLM pool** (`llmpool/`, `agent/provider.go`): ordered profile chains
  with failover and per-intent round-robin rotation (A, B, A, B by dispatch
  order). Health registry: hard failures trip the breaker immediately,
  soft failures need repeats, cooldown ladder grows, a pass resets. No
  failover after the first streamed token is emitted, none on caller
  cancel; members whose context window is smaller than the estimated
  request are skipped; when every member is tripped the head is still
  probed. Every call is recorded (`llmrec/`) for token accounting.
- **Enrich** (`enrich/`): async DNS/HTTP enrichment of new assets (title,
  status, tech) with cooldowns, through the same egress governor.
- **Operator handoff:** on an environmental block (sandboxed network,
  missing binary/key, TLS interception), the agent stops after the first
  failure — no flag-shopping or tool-switching — and calls `ask_operator`
  with exactly one copy-pastable command and what to paste back.
- **Deadline contract:** an agent must not start a command whose timeout
  can outlive the task deadline; near the deadline it shifts from
  discovery to consolidation.

## Platform security (of ARTEX itself)

Secrets encrypted at rest (above); first-run `/setup` claim requires a
one-time token printed at startup; UI binds loopback by default; login
rate limit 10/min/IP with `ARTEX_TRUSTED_PROXIES` CIDR handling that reads
`X-Forwarded-For` from the right past trusted hops only; server env scrub
for LLM keys at startup (with the documented `/proc/<pid>/environ` limit);
releases carry SHA256SUMS + CycloneDX SBOM + GitHub provenance attestation.
The CHANGELOG's "Known issues" section is unusually candid (build context
can include local secrets in a build-stage layer, recon tools pinned but
not checksum-verified, sqlmap tracking its default branch) — treat it as
part of the decompile surface, not marketing.

## Bundled skills (`skills/`)

Ten playbooks the agents load: `api-recon` (route-map extraction incl.
runtime harvesting), `api-security` (BOLA/BFLA/JWT/OAuth/GraphQL checklists),
`bbot-recon`, `codeaudit` (white-box forward tracing, VulnHunter-derived),
`edr-bypass-re`, `patch-diff`, `php2shell`, `playwright-cli`, `rea`
(reverse-engineering assistant), `scopesentry` (ScopeSentry MCP sync into
the asset graph).

## RSI contract (`RSI.md`)

Three loops: A within-task (exists), B cross-task lessons (shadow mode;
`ARTEX_RSI_LESSONS=off|shadow|inject`; a lesson needs cue + action +
evidence, status stays `candidate`, injection is a code-owned data-framed
block), C on-the-harness self-modification (**not built** — contract only:
benchmarks adopt, live tasks only propose; owner approval per adoption;
gates/scope/injection defenses are immutable surfaces; one-step rollback;
flag off by default). A decompile must report Loop C as a contract, not a
capability.

## Unknowns (bounded-search disclosure)

- norma SDK internals (`BaaRai/Norma` v0.4.11) were not decompiled in this
  pass — agentcore's harness, permission modes, and NOA live there; ARTEX
  facts above that depend on them are cited to the ARTEX call sites only.
- The Next.js frontend was inventoried, not line-audited; UI claims rest on
  the Go API + schema.
- No live instance was run; runtime behaviour claims are static-trace
  claims against code + tests (~1,269 test functions were used as the
  executable specification where prose and code disagreed).

## Decompile method used (reusable)

1. `go.mod` first: module path, Go version, the `replace` that reveals the
   forked SDK. 2. `db/schema.sql` as the data-model ground truth before
   reading logic. 3. `cmd/*/main.go` for wiring, then the engine loop.
   4. The prompt/tool catalogs (`promptcatalog.go`, `toolcatalog.go`) —
   in an agent system the seeded prompts *are* the behaviour spec; separate
   editable bodies from code-owned tails. 5. The gate packages (`guard/`,
   `intercept/`) read as a pipeline in hook order. 6. Tests as executable
   spec, especially `*_seed_test.go` (seeded prompts/rules are pinned by
   tests) and the known-issues prose in README/CHANGELOG.
