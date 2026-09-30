# AKR Baseline — Selbst gemessen, vor der Synthese

**Quelle:** `/srv/aki/kernel`, Commit `ba3a6f4c52d0b02e2cdb9504f68df60665c4d9d5` (2026-09-30, `fix(phase-140): three gates read measu…`)

Zweck: Die drei Fremdsystem-Reports werden von Agenten geschrieben, die Aki gebaut haben. Ohne
gemessenen AKR-Ist-Stand würden die Adopt/Adapt/Reject-Entscheidungen (§13 des Rechercheplans) nur
das bestätigen, was bereits da ist. Diese Datei ist der Kontrollpunkt — jede Aussage über "AKR hat
das schon" muss hier auf eine Datei:Zeile zurückführbar sein.

## 1. Wo der Turn lebt

| Element | Ort | Beleg |
|---|---|---|
| Loop-Einstieg (Plugin) | `plugins/agent/src/handle_chat.rs:4` | `pub(crate) async fn handle_chat` |
| Iterationsschleife | `plugins/agent/src/handle_chat.rs:429` und `:702` | zwei `loop {` |
| Prompt-Montage | `plugins/agent/src/turn/prompt.rs` | `push_str`-Kette ab `:218` |
| Turn-Budget | `plugins/agent/src/turn/budget.rs:264` | `with_round_limits`, Clamp `.max(1).min(50)` |
| Budget aus Config | `plugins/agent/src/turn/budget.rs:79-90` | `max_llm_rounds`, `max_tool_rounds` |
| Wall-Clock-Budget | `plugins/agent/src/turn/budget.rs:128` | `turn_budget_secs`, Default 120, geklemmt 10..3600 |
| Spill | `plugins/agent/src/spill.rs:73` | `pub fn spill(session_id, tool, text)` |
| LLM-Idle-Deadline | `plugins/llm/src/deadline.ts:26` | `idleMs` + `armIdle()`, Abbruch `"idle"` |
| Review-Loop | `plugins/agent/src/review.rs` (516 Z) | Skill-Review-Fork |

### Prompt-Tier-Modell (nachträglich gemessen, ersetzt die frühere Annahme "unbekannt")

`plugins/agent/src/turn/context_policy.rs` modelliert den Prompt in fünf Sektionen:

| Sektion | Rolle | Quelle |
|---|---|---|
| `Stable` | unantastbarer Prefix, wird **nie** gekürzt | `context_policy.rs:25` |
| `Skills` | prozedurales Wissen | `handle_chat.rs:566` |
| `Memory` | injiziertes Memory | `handle_chat.rs:567` |
| `Errors` | Error-Reconciliations-Historie | `handle_chat.rs:568` |
| `Notices` | dynamische Hinweise | `handle_chat.rs:569` |

`CUT_ORDER` (`context_policy.rs:88`) opfert in dieser Reihenfolge:
**Notices → Errors → Memory → Skills**. `Stable` ist nicht in der Liste, wird also zuletzt
und nur mit dem Marker `budget/over-stable` angefasst (`:126-133`) — der Überhang verschwindet
also nicht still. Zählung in **Chars, nicht Bytes** (`handle_chat.rs:571`, Test
`budget_counts_chars_not_bytes`), Dedupe gegen wiederholte Bullet-Zeilen ab 12 Zeichen
(`context_policy.rs:161` + `DEDUPE_MIN_LINE_CHARS:147`), Degradation bei Timeout
(`degrade:242`). Das Ergebnis geht als `policy` in das bestehende `context.inject`-Event —
die Kürzung ist damit selbst ein beobachtbares Event, kein stiller Seiteneffekt.

**Konsequenz für die Synthese:** AKR hat kein fehlendes Compaction, sondern ein *deterministisches
Prioritäts-Budget* für den dynamischen Prompt-Tail. Das ist eine andere Antwort auf dieselbe
Frage als die der drei Fremdsysteme — hier nicht als Lücke behandeln.

**Befund:** AKR hat Budget-Klammern **zwei** (Runden UND Wanduhr) plus Idle-Deadline auf dem
LLM-Stream. Das ist mehr als bei den beiden anderen Systemen an dieser Stelle messbar war. Wer
„Budget" als fehlende AKR-Lücke behauptet, hat nicht gemessen.

## 2. Store ist der Guard, nicht der Loop

| Element | Ort |
|---|---|
| Schreib-Guard | `src/store/mod.rs:87` `mutate<T>` |
| Lese-Guard | `src/store/mod.rs:120` `check_read(ctx, ns)` |
| Daten-Mutation | `src/store/mod.rs:141` `mutate_data<T>` |
| CAS / Version | `src/store/mod.rs:677` `set_if` |
| Capability-Requirements | `src/gateway/routes/rev.rs:390`, `src/gateway/mod.rs:470` |

Drei Aufrufer von `check_capability_requirements` (REST `rev.invoke`, intern `invoke.rs:132`,
NATS-Einstieg) — die Gate-Reihenfolge ist für alle drei Ingressen geschlossen.

## 3. Was AKR vergleichsweise DÜNN ist (ehrliche Lückenliste)

Das sind die Stellen, an denen die Fremdsysteme möglicherweise weiter sind. Diese Liste ist bewusst
vorformuliert, damit die Synthese sie bestätigen oder verwerfen muss:

1. **Event-Log im Kernel dünn:** `src/event_log/envelope.rs` 197 Z + `mod.rs` 84 Z = 281 Z
   gesamt, `src/event_bus/mod.rs` 19 Z. Die eigentliche Session-Historie liegt im Session-Plugin
   (`plugins/session/src/event_log.rs`, 2278 Z). Der Kernel-Event-Log ist also kein vollständiges
   Source-of-Truth — der Satz „AKR hat einen append-only Event-Log als Source of Truth" ist so
   nicht belegbar.
2. **Compaction der Session-Historie:** Der dynamische Prompt-Tail ist abgedeckt (siehe
   Prompt-Tier-Modell oben, `CUT_ORDER`). Für die *Chat-Historie* selbst ist jedoch kein
   Verdichtungs-Modul gefunden: `src/kernel/subscribers/usage_watch.rs` misst Wachstum über
   `run_id`, verdichtet aber nicht. `plugins/session/src/event_log.rs` (2278 Z) ist der
   Session-Historie-Speicher. Ob und wo dort Historie gekürzt wird, ist offen.
3. **Doom-loop-Erkennung:** keine dedizierte Stelle gefunden (drei Stellen suchen nach
   `doom|repeat` → nur Rundenbudgets). dsh hat `repeat-tool-reminder` mit Nudge bei 3/5/8
   identischen Calls — das hat AKR nach aktuellem Stand nicht.
4. **Memory-Arten:** Store-Namespaces existieren, aber die Trennung „immer injiziert" vs.
   „Retrieval" vs. „Event-History" vs. „Skills" ist im Code nicht als Typ-System erkennbar.
5. **Subagenten:** `plugins/agent/src/` hat kein Subagent-Modul. Nur `agent.dispatch` über den
   Gateway-Pfad.

## 4. Anti-Bias-Regel für die Synthese

Eine Adopt/Adapt/Reject-Entscheidung ist nur gültig, wenn sie entweder
- auf eine **Lücke in Abschnitt 3** zielt, oder
- eine **bewusste AKR-Entscheidung** umkehrt, die in einer Spec festgehalten ist.

Alles andere ist Rauschen. Wenn ein Subagent „AKR sollte X adoptieren" schreibt und X bereits in
Abschnitt 1 oder 2 belegt ist, wird der Befund verworfen.
