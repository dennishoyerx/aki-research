# Aki Research Findings

Derived from `reports/hermes.md`, `reports/deepseek-harness.md`, `reports/opencode.md`, measured
against `reports/aki-baseline.md` (AKR @ `ba3a6f4`).

**No roadmap phases.** Findings and decisions only.

**Method note:** the three reports were written by agents that also built Aki. Every adopt/adapt/
reject verdict below was therefore re-checked against AKR source. Five of the initially assumed gaps
were disproved that way; they are recorded in the baseline as disproved, not as work items.

---

## Verified corrections to earlier working notes

| Claim | Reality | Evidence |
|---|---|---|
| Spill threshold is 8 KiB | **50 000 UTF-8 bytes**; the 8192 figure is the separate tool-result pruner | `spill-policy/src/index.ts:1-30` |
| AKR has no compaction | Mechanic **and** model-triggered exposure exist; the deterministic trigger is what is missing | `event_log.rs:318-486`, `main.rs:1849` |
| AKR has no doom-loop detection | Notice at 3/5, hard abort at 8, abort logged as event | `handle_chat.rs:946-1004` |
| AKR has no spill skip-list | `SPILL_SKIP = ["file.read","file.patch","file.diff"]` | `spill.rs:17` |
| AKR has no goal CAS | `advance(conv,id,revision)` with `goal_rev != r.revision` guard | `goal_round_driver.rs:157-158` |

---

## Findings

### F-1 The only hard gap: recursion depth
AKR has `agent.dispatch` but no `max_spawn_depth`, no recursion bound, no subagent budget quota
(grep over `src/` and `plugins/agent/src/` finds nothing). Hermes caps at `max_spawn_depth: 1`, dsh at
a 256 round cap. A dispatch that dispatches can recurse without bound.
**Decision: ADOPT** a kernel-side depth guard on the dispatch path. It is the one place where all
three systems protect the kernel and AKR does not.

### F-2 Compaction is model-triggered, not deterministic
AKR can compact (`event_log.rs:318`) and exposes it (`main.rs:1849`), but the summary text and the
decision to compact both come from the model. No threshold, no turn hook.
**Decision: ADAPT** — keep the model's authority over *what* to summarise, move the *when* into the
deterministic path using the existing `ContextReport` (`context_policy.rs`), which already measures
section sizes.

### F-3 Spill policy and spill mechanics are fused
dsh's policy registers no service and owns no storage; preview and file live behind seams. AKR's
`spill.rs` holds decision, preview, file handling and cleanup.
**Decision: ADAPT** — split into a decision (`should_spill(text, budget) -> Option<Locator>`) and a
backend. The skip-list and the `None`-means-caller-truncates contract stay unchanged.

### F-4 Two spill thresholds are better than one
dsh caps the model-facing result at 50 000 bytes and the durable log copy at 8192 chars. AKR has one
budget (`DEFAULT_BUDGET = 2400`, `spill.rs:19`).
**Decision: ADAPT** — separate context relief from log size. The current single value cannot serve
both purposes without compromise.

### F-5 Event-bound invariants are worth adopting
dsh validates the event log with an independent fold and reports the failing `seq`
(`goal/src/invariant.ts`). AKR's kernel event log is thin (281 Z) while session history lives in the
session plugin — the derived history there is not independently verifiable.
**Decision: ADOPT** for the session plugin's derived history.

### F-6 Fresh-budget-per-subagent must be rejected
Hermes gives subagents a fresh `IterationBudget`; the docstring states the parent's cap can be
exceeded in total. That is correct for a process-based agent and wrong for a kernel that promises
deterministic bounds.
**Decision: REJECT** explicitly and record the reason, so it does not get re-imported from Hermes
later.

### F-7 Doc/code drift is its own failure class
`.env.example:447` says `CONTEXT_COMPRESSION_THRESHOLD=0.85`; the code default is 0.50
(`context_compressor.py:2692`). Following the doc compresses at 85 % instead of 50 %.
AKR has no equivalent measured drift, but its own skill references encode thresholds in prose.
**Decision: INVESTIGATE** — a gate that asserts documented defaults equal code defaults.

### F-8 Exposure is a permission question, not a performance question
Only AKR makes tool exposure resolve through ACL (`tool_info.rs`, `check_read`/`mutate`). Hermes hands
over all core tools every turn and even mutates the set on MCP reconnect
(`mcp_tool_agent.py:86`, `:234`). AKR's position is the strongest of the four and should be defended.

### F-9 Learning loops differ by generation
Hermes has an LLM-driven skill/memory loop with a curator (1212 Z). dsh has durable logged state but no
skill learning. OpenCode has nothing. AKR has a skill-review fork.
**Decision: NO CHANGE** — AKR is ahead of two of three. The gap is review cadence, not existence.

### F-10 Two runtimes is a trap, not a pattern
OpenCode ships V1 (`src/session/*`) and V2 (`packages/core/src/session/*`) simultaneously; V1 remains
the production path and the defect list includes V2 TODOs in runtime code.
**Decision: REJECT** the parallel-runtime approach. AKR's plugin-based extension (agent, session, llm
as separate plugins) achieves the same modularity without two authoritative paths.

---

## Answers to the twenty questions

**1. What should Aki's agent loop be?** A phase pipeline in the agent plugin, with the kernel owning
only what must be deterministic: budget clamp, depth guard, ACL, store writes. Hermes' verdict
interface (`conversation_loop.py:1480`) is the closest reference; AKR already has two `loop {` sites
that could become phases.

**2. Kernel or plugin?** Kernel: dispatch depth, store guards, preconditions, budget clamp. Plugin:
prompt assembly, tool execution, compaction content, learning. The current split is already correct;
the missing kernel piece is the depth guard.

**3. What is session state?** The append-only log in `plugins/session/src/event_log.rs`. It is
authoritative today; the kernel's own event log is not the session history.

**4. What is memory?** Store namespaces plus injected prompt sections. Currently not distinguished by
type. Hermes' four-way split (injected / retrieval / history / skills) is the cleanest available
taxonomy and costs nothing to adopt as documentation.

**5. What is retrieval?** Store search (`store/mod.rs`, `search` handle). Note that
`DiskBackend::query` ignores filters — retrieval quality is limited by that, not by design.

**6. What is context?** The five sections in `context_policy.rs` plus session history. Already a
tiered model with a guaranteed-stable prefix.

**7. What is a skill?** A store object under `skills.*` with a revision, gated on capability. Its
review loop already exists (`review.rs`).

**8. What belongs in the prompt always?** The `Stable` section only. Everything else is subject to
`CUT_ORDER`. This is already enforced.

**9. What should be retrieved on demand?** Memory (Qdrant), skills, session search — all three are
already store-backed rather than prompt-injected.

**10. How should tool exposure work?** As it does: resolve per intent, gate by ACL. This is the
strongest of the four systems. Defend it; do not "optimise" it into a performance-driven reduction.

**11. How should compaction work?** Model decides the content, kernel decides the moment. F-2.

**12. How does Aki prevent doom loops?** It already does, at two levels plus a third possible
(text-side, `repetition_guard`-style). Adding text-side repetition detection is the open item.

**13. How should subagents work?** Dispatch with an inherited depth counter and a shared budget —
explicitly *not* Hermes' fresh-budget model. F-1, F-6.

**14. How should learning work?** As it does, with review cadence rather than new mechanism. F-9.

**15. How does Aki stay provider-independent?** Through the `llm` plugin boundary and the
`auxiliary.*` config split. All three reference systems have provider-specific breakage in their
prompt paths; AKR's isolation is a real advantage.

**16. What must be deterministic?** Budget clamp, depth guard, ACL/preconditions, store writes,
prompt section budget, tool-loop abort. Everything else may be model-decided.

**17. What may the model decide?** Compaction content, tool choice, whether to delegate, what to
remember as a candidate, and whether to emit a skill mutation proposal.

**18. Where does Aki need hard kernel guards?** Dispatch depth (missing), store mutation, capability
preconditions, tool-loop abort, prompt budget.

**19. Which state transitions need event/revision safety?** Goal transitions (already CAS-guarded),
store writes (`set_if`), and — if F-2 is adopted — the compaction marker.

**20. What should Aki explicitly not take?** Fresh-budget subagents (F-6), parallel runtimes (F-10),
the 10k-line compression module pair, single huge modules (`context_compressor.py` 5673 Z,
`auxiliary_client.py` 8255 Z) — AKR's `goal_round_driver.rs` at 21983 Z is already the counter-example
to over-splitting, so this is a judgement call, not a rule.

---

## Open questions

- Whether AKR's session-history derived view can be made independently verifiable (F-5) without
  duplicating the event log.
- Whether the prompt section budget should drive compaction timing directly (F-2) or stay separate.
- Whether text-side repetition detection is worth adding on top of the existing tool-side detector.
- Whether the review cadence for skills is measurably too low — no metric exists for this.

---

## Kurzfassung für Dennis

Drei Systeme, eine Matrix, und die Matrix sagt: **AKR ist in vier Bereichen stärker und in genau einem
schwächer.** Stärker bei Turn-Budgets (Hermes defaultet auf `sys.maxsize`, dsh hat gar keins), bei
Doom-Loop-Erkennung (3/5 plus Hard Abort bei 8), bei Tool-Exposure als Berechtigungsfrage statt
Performancefrage, und beim Lern-Loop. Schwächer bei genau einer Sache: **AKR hat keine Rekursionsgrenze
für Subagenten.** `agent.dispatch` kann dispatchen, das dispatchen kann dispatchen — kein
`max_spawn_depth`, keine Quote, kein Limit. Das ist die einzige Stelle, an der alle drei anderen
Systeme den Kernel schützen und AKR nicht, und es ist eine Zeile Gate im Dispatch-Pfad.

Fünf angenommene Lücken waren falsch und sind als widerlegt dokumentiert: Compaction gibt es (nur
modellgetriggert), Doom-Loops werden erkannt, die Spill-Skip-Liste existiert, Goal-CAS existiert, und
die Spill-Grenze ist 50.000 Bytes statt 8 KiB. Ein Suchfehler meinerseits — ich hatte nach
`compaction|summar` gesucht statt nach `compact` im Event-Log — hat die erste falsche Annahme
erzeugt. Genau dafür war der Anti-Bias-Pass da, bevor sie in eine Roadmap wandert.

Übernommen wird: Rekursionsgrenze, Event-Invariante für die abgeleitete Session-Historie, Trennung von
Spill-Entscheidung und Spill-Mechanik, Doppelgrenze 50k/8192. Abgelehnt wird ausdrücklich: Hermes'
frisches Budget pro Subagent und OpenCodes Doppelruntime. Beides steht jetzt begründet im Repo, damit
es nicht später über eine Hermes-Spec wieder einfließt.
