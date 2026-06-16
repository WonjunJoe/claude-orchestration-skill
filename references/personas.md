# Personas — index

Each persona is its own prompt template under `references/personas/`. **Don't load them all into the orchestrator's context.** When you dispatch a worker, point it at its own file: the worker reads `references/personas/<persona>.md` + `references/personas/_universal.md` and follows them. The orchestrator only picks the right persona and passes the path — the prose lives with the worker that uses it, not in the orchestrator's head.

Every persona ends with **the 5-field handoff format**. That's intentional — the orchestrator parses these fields after each return. Don't let workers improvise their own report shapes.

| Persona | File | Role |
|---|---|---|
| Domain Audit Worker | `personas/01_domain-audit-worker.md` | Map the codebase before changes (read-only) |
| Research Worker | `personas/02_research-worker.md` | Investigate external knowledge — docs, specs, prior art |
| Feature Implementer | `personas/03_feature-implementer.md` | Build NEW behavior → one commit |
| Refactor Implementer | `personas/04_refactor-implementer.md` | Restructure WITHOUT changing behavior |
| Fix Implementer | `personas/05_fix-implementer.md` | Fix a broken behavior (TDD) |
| Functional Verifier | `personas/06_functional-verifier.md` | *Does it work?* |
| Architecture Verifier | `personas/07_architecture-verifier.md` | *Is it well-built?* |
| Black-User E2E Validator | `personas/08_black-user-e2e-validator.md` | *Does a clueless user succeed?* |
| Design Verifier | `personas/09_design-verifier.md` | *Is it 1-tier quality?* |
| _(shared)_ | `personas/_universal.md` | Rules every dispatch applies — load alongside any persona |

When you dispatch, fill in the task-specific details (file paths, the exact change, the exact verification criteria) in the worker's prompt. Reuse the persona's identity and procedural rules verbatim — the consistency is what makes the loop work across many invocations.

## Implementer personas (three types, by intent)

The Implementer role is split into three personas by intent. The choice affects mindset, procedure, and which verifier is the toughest critic. Pick the right one before dispatch:

- **Feature Implementer** — adds NEW behavior the system didn't have. New endpoint, new screen, new business rule. The default for "build me X."
- **Refactor Implementer** — restructures code WITHOUT changing behavior. DRY extraction, consolidation, deepening, dead code removal, renames. If the diff would change what the system does, it's not a refactor.
- **Fix Implementer** — fixes a broken behavior. Bug report, failing test, regression, wrong calculation. TDD discipline: failing test first, then green.

All three share the same `[worker]` dispatch tag with a subtype: `[worker:feature]` / `[worker:refactor]` / `[worker:fix]`.

## Notes on persona reuse

These templates are **starting points**, not straitjackets. When you dispatch:

- Add task-specific context (which files, which environment, which reference) in the worker's prompt.
- Trim irrelevant procedure steps. A pure CSS change doesn't need a SQL audit.
- Keep the **identity** and **5-field handoff** stable. That's what makes the loop composable.

If you find yourself dispatching the same persona with the same edits over and over, that's a sign this skill should grow another reference file with that variant. Update the skill rather than re-improvising it each session.
