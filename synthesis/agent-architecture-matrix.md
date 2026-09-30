# Cross-System Architecture Matrix — Hermes / dsh / OpenCode / Aki

**Sources (pinned):**
- Hermes Agent `NousResearch/hermes-agent` @ `6ec05205a943cf813bd56c3c79d64bcf922dac67`
- DeepSeek Harness `deepseek-ai/deepseek-harness` @ `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8`
- OpenCode `anomalyco/opencode` (dev) @ `97a86b7677c38e7a5a9cc2a0f6849cfb4f2ce462`
- Aki / AKR `/srv/aki/kernel` @ `ba3a6f4c52d0b02e2cdb9504f68df60665c4d9d5`

**Rules obeyed:** no scores, no "X is better". Every cell carries a source reference or the literal
`nicht gemessen`. Aki cells cite AKR file:line, never an AKR spec.

---

## Matrix

| Area | Hermes | DeepSeek Harness | OpenCode | Aki (AKR) |
|---|---|---|---|---|
| **Agent loop** | 13-phase pipeline, verdict interface, `_LoopState` with ~45 fields (`conversation_loop.py:1480`, `:1385`) | No central loop — composition of Cordis plugins; three nested loops (step / round / turn) | **Two runtimes in parallel**: V1 `SessionPrompt.run` (`prompt.ts:1081-1341`), V2 `SessionRunner.run` (`runner/llm.ts:390-413`) | Loop in the agent plugin, not the kernel: `handle_chat.rs:4`, two `loop {` at `:429`/`:702` |
| **Turn/step budget** | `IterationBudget` per **agent instance**; default `max_iterations` is `sys.maxsize` (`iteration_budget.py:25`, `run_agent.py:268`) | **None in the default agent loop** — no step cap, no wall-clock cap | Token-budget based; not measured in detail | **Two clamps**: rounds + wall clock (`turn/budget.rs:264`, `:128`) |
| **Loop phases / verdict** | Uniform verdict interface; latched fields cannot be reset (`conversation_loop.py:1477`) | not measured | not measured | not measured |
| **Prompt assembly** | Three tiers: stable / context / volatile; per-conversation caching declared sacred (`AGENTS.md:19`) | Assembly *is* the plugin composition; order = registration order | V1 explicit ordering; V2 registry-based (`2.2`) | Five sections with priority cut order (`turn/context_policy.rs:88`) |
| **Tool exposure per turn** | **All core tools every turn** — `tools_for_api = agent.tools`, unconditional (`turn_request_assembly.py:196`); MCP reconnect mutates it mid-session (`mcp_tool_agent.py:86`, `:234`) | All tools per agent scope; `ptc` mode reduces to `run_code` | Dynamic selection exists but **not context-aware** (`5.3`) | Per-intent schema resolution + ACL (`tool_info.rs:79-131`) |
| **Memory kinds** | Four distinct: injected (`MEMORY.md`/`USER.md`), retrieval (provider + session search), session history, skills | **No standalone memory system** — role is split across 4 mechanisms | AGENTS.md/skill, session history, no memory write | Store namespaces exist; the four kinds are **not a type system** |
| **Memory write** | LLM decides (`learn_prompt.py`, `curator.py` 1212 Z) + security gate | Deterministic only; model can only query | **None** — no learning loop at all | Skill-Review-Fork (`plugins/agent/src/review.rs`) |
| **Prompt cache tiers** | Explicit stable prefix, cached markers (`turn_request_assembly.py:196` region) | not measured | Context-Epoch (`packages/core/src/session/context-epoch.ts`) | not measured (no context epoch) |
| **Session state** | SQLite `hermes_state*.py` (31 modules), 13 `CREATE TABLE` | Append-only event log = single source of truth; FTS5 is a **disposable read model** | Message store; V2 has projector (`projector.ts` 455 Z) | Kernel event log thin (281 Z); history in session plugin (`event_log.rs` 2278 Z) |
| **Compaction** | Threshold default **0.50** in code (`context_compressor.py:2692`) vs **0.85** in `.env.example:447` — doc/code drift | `compaction-basic` (`index.ts:148`, `:180`) | V2 compaction (`compaction.ts` 248 Z); documented auto-compaction reorder bug | Mechanic exists (`event_log.rs:318-486`), exposed as model tool (`main.rs:1849`) — **no deterministic trigger** |
| **What survives compaction** | Protect-first 3 / protect-last 20 (`context_compressor.py:2692`) | not measured | not measured | `Stable` section never truncated; `budget/over-stable` marker instead of silent loss (`context_policy.rs:126-133`) |
| **Doom-loop detection** | Three levels: text (`repetition_guard.py:49`, `:137`), tool dedup + delegate cap, thinking-only (`run_agent.py:1142`) | Advisory nudge at `[3,5,8]`, user interjection resets (`repeat-tool-reminder/src/index.ts:29`, `:226-230`) | Session-loop bug that killed the session (`10.2`) | **Already stronger**: 3/5 notice + hard abort at 8 (`handle_chat.rs:946-1004`) |
| **Tool-result handling** | Compression after tool results (`turn_tool_round.py:18` import) | **Spill to artifact**: full text to file, model gets head/tail + locator (`spill-policy/src/index.ts:1-30`) | Truncation; **no spill** | Spill with skip-list (`spill.rs:17` `SPILL_SKIP`) |
| **Subagents** | Kind threads in the same process; **fresh budget, no hierarchical share** (`delegate_tool.py:246`) | not measured | `@`-mention → task tool | `agent.dispatch` exists; **no depth limit** |
| **Recursion guard** | `max_spawn_depth: 1` in delegation config | Round cap 256 | not measured | **absent** — no `max_spawn_depth` anywhere in `src/` or `plugins/agent/src/` |
| **Persistence** | SQLite + WAL, persist-before-execute durability invariant (`turn_tool_round.py:55-59`) | Append-only log; rebuildable FTS5 | Message persistence; crash repair | Store `mutate`/`check_read`/`set_if` (`store/mod.rs:87`, `:120`, `:677`) |
| **Resume** | Rebuilds context from DB; partial tool calls not measured | Replay log | V2 `context-epoch.ts` | not measured |
| **Observability** | 2750 issue refs in code; `except Exception: pass` **156×** | Event log is the trace | not measured | `usage_watch.rs` measures growth; events on every tool round |
| **Extensibility** | Plugin via `$HERMES_HOME/plugins/`, tools registry auto-discovery | Capability seam: definition + provider + consumer | Plugin system + ACP + server layering | 3-layer capability → provider → implementation, intent-alias |

---

## Where the three systems solve the same problem three different ways

These are the rows that matter for Aki. Confirmations are listed briefly.

### 1. Bounding a runaway turn

- **Hermes**: budget object per agent instance, but the default cap is `sys.maxsize` — the real limit comes from config/gateway. Subagents get a *fresh* budget, so total work can exceed the parent's cap.
- **dsh**: no default budget at all.
- **OpenCode**: token-budget driven.
- **AKR**: rounds *and* wall clock, both clamped.

Divergence: nobody agrees on where the bound belongs. Hermes puts it in an object that can be
defaulted to infinity; dsh omits it; OpenCode ties it to tokens; AKR splits it across two dimensions.
The interesting part is that AKR is the only one where the bound is guaranteed present.

### 2. Keeping tool output out of the context

- **Hermes**: compress after tool results (lossy, summarises).
- **dsh**: spill to an addressable artifact, keep the full text, give the model a locator (lossless).
- **OpenCode**: truncate.
- **AKR**: spill, with a skip-list preventing read→spill→read.

Divergence: dsh is the only lossless one — the content is still reachable by path. AKR already
adopted this shape. The gap is dsh's *separation*: the policy registers no service and owns no
storage, while AKR's `spill.rs` is one module holding both.

### 3. Stopping a looping agent

- **Hermes**: three independent detectors, thresholds calibrated against production data (80k–350k chars).
- **dsh**: advisory only, never vetoes, user interjection resets.
- **OpenCode**: a bug where the loop *killed the session*.
- **AKR**: notice at 3/5, hard abort at 8, abort is a logged event.

Divergence: advisory versus vetoing. dsh deliberately never vetoes; AKR vetoes at 8. Hermes detects on
the *text* side as well as the tool side, which none of the others do.

### 4. Deciding what the model sees

- **Hermes**: everything, every turn, plus mid-session mutation on MCP reconnect.
- **dsh**: everything per scope, one execution mode reduces to a single tool.
- **OpenCode**: dynamic selection, but not context-aware.
- **AKR**: per-intent resolution plus ACL — the model cannot see what the caller may not use.

Divergence: only AKR makes exposure a *permission* question rather than a *performance* question.

### 5. Recording state

- **Hermes**: canonical messages array in SQLite, near event-sourcing without the event log.
- **dsh**: append-only event log, derived state explicitly disposable, invariants validate events by `seq`.
- **OpenCode**: message store, and during a migration two runtimes coexist.
- **AKR**: thin kernel event log, real history in the session plugin.

Divergence: dsh's invariant idea — validate the log with an *independent* fold and report the
failing `seq` — is the one AKR does not have and arguably should.

---

## Per-system summary

### Hermes
- **Strength**: the loop is a phase pipeline with a uniform verdict interface; latched fields protect
  an overflow-recovery arming from being reset by a single retry (`conversation_loop.py:1477`).
- **Trade-off**: everything is in one Python process — `context_compressor.py` 5673 Z,
  `auxiliary_client.py` 8255 Z, 156 silent `except Exception: pass`.
- **Interesting idea**: restart-counter separate from the retry-counter, learned from a production
  runaway (`conversation_loop.py:1420-1424`).
- **Known weakness**: doc/code drift (`0.85` vs `0.50`); `SOUL.md` and `config.yaml.example` do not
  exist in the repo; evals check mechanics only, never answer quality.
- **Relevance for AKR**: reject the fresh-budget-per-subagent model; adopt config-beats-model for
  every budget a model can influence.

### DeepSeek Harness
- **Strength**: the capability seam is a *rule*, not a pattern — enforcement sits where the decision
  is made, and the spill policy owns nothing.
- **Trade-off**: no turn budget means a runaway turn is bounded only externally.
- **Interesting idea**: event-bound invariants that report the failing `seq`
  (`goal/src/invariant.ts`).
- **Known weakness**: spill is a silent no-op unless configured; local filesystem storage.
- **Relevance for AKR**: split the spill *decision* from the spill *mechanics*.

### OpenCode
- **Strength**: a spec-first core (`specs/session.md`) that documents intent before code.
- **Trade-off**: two runtimes coexist; V1 is still the production path.
- **Interesting idea**: context-epoch as an explicit cache-coherency concept.
- **Known weakness**: nine documented defects, including double auto-compaction caused by reorder
  and an MCP SSE reconnect loop.
- **Relevance for AKR**: do not build V2 before V1 is stable — that is the trap this repo demonstrates.
