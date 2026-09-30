# aki-research

Architecture research for Aki / AKR: how agent runtimes are actually built.

## Systems under study

| System | Repo | Anchor commit |
|---|---|---|
| Hermes Agent | NousResearch/hermes-agent | `6ec05205a943cf813bd56c3c79d64bcf922dac67` (2026-09-30) |
| DeepSeek Harness | deepseek-ai/deepseek-harness | `c389f96bf3a9b6807cb71ed6bdad5849be0df6d8` (2026-09-08) |
| OpenCode | anomalyco/opencode (dev) | `97a86b7677c38e7a5a9cc2a0f6849cfb4f2ce462` (2026-10-01) |

AKR under study: `/srv/aki/kernel` @ `ba3a6f4c52d0b02e2cdb9504f68df60665c4d9d5`.

## Research areas

1. Agent loop  2. Prompt assembly  3. Memory  4. Context management  5. Tools
6. Agent state  7. Subagents / delegation  8. Learning / self-improvement
9. Persistence / resume  10. Reliability / failure modes  11. Cross-system matrix
12. Aki findings (adopt / adapt / reject / investigate)

## The question behind the research

Not "which is best" and not "how do we copy them". Instead:

> Which agent problems have others solved, which problems did they acquire in the process, and
> which architectural decisions are recognisable from that?

Differences are more valuable than confirmations. When three systems solve the same problem three
different ways, that is more informative for Aki than three confirmations of one architecture.

## Status

- [x] Baseline: AKR measured state (`reports/aki-baseline.md`)
- [ ] Hermes report
- [ ] DeepSeek Harness report
- [ ] OpenCode report
- [ ] Cross-system matrix (`synthesis/`)
- [ ] Aki findings (`synthesis/aki-findings.md`)
