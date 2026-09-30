# aki-research — Agent Runtime Architecture Research

Private research repository. Purpose: document, with evidence, how leading agent systems
(Hermes Agent, DeepSeek Harness, OpenCode) actually build their runtime, and what that means
for Aki / AKR.

## Evidence rule (non-negotiable)

Every important claim carries a reference: `repo + commit SHA + file + symbol/line`.
Marketing text and README prose are not technical evidence.

Priority: source code > official architecture doc > official README/docs > issue/PR >
commit history > maintainer statement > external analysis.

## Layout

- `reports/` — one report per system
- `reports/aki-baseline.md` — AKR's own measured state, written FIRST, before any verdict
- `synthesis/` — cross-system matrix, Aki findings, answers to the 20 Aki questions

## Anti-bias rule

The reports are written by agents that also helped build Aki. Therefore:

1. Any adopt/adapt/reject verdict must target a gap listed in `reports/aki-baseline.md`, or
   deliberately reverse a decision that is documented in an Aki spec.
2. Verdicts about AKR cite AKR file:line, never an AKR spec.
3. "Three systems agree" is weaker evidence than one system having measured a real failure.

## Language

Specs and research docs are written in English. `reports/aki-baseline.md` is German because it is
a working note for the repository owner.
