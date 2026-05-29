# claude-orchestration-skill

A [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill for running complex, multi-step engineering work as a small adversarial team of agents instead of one linear pass.

The main Claude Code session becomes an **orchestrator**: it plans, dispatches fresh-context workers, runs independent validators against every commit, and loops `verify → fix` until the work is provably ready. Built from real production sessions — see [`references/examples.md`](references/examples.md) for a walkthrough of a 4-round, 25-commit redesign cycle.

## The pattern

```
Phase 1 — Audit (parallel, read-only)
   ↓ findings + fix proposals
Phase 2 — Implement (serial, fresh worker per feature)
   ↓ commits + 5-field structured handoff
Phase 3 — Verify (parallel, independent context, adversarial)
   ↓ PASS / PASS_WITH_CONCERNS / NEEDS_REVISION / FAIL
Phase 4 — Loop
   ↓ NEEDS_REVISION → fix worker with critique pasted verbatim → re-verify
   ↓ PASS → ship + report
```

Each agent gets a **fresh context** and a **narrow remit**. The worker implementing a fix cannot sneak its own design opinions past a validator who has never seen the code before. The orchestrator stays clean — it doesn't try to be implementer + critic + tester at once.

## Personas

Nine reusable agent personas, each with a full prompt template in [`references/personas.md`](references/personas.md):

| Persona | Role | When to dispatch |
|---|---|---|
| **Domain Audit Worker** | Map territory *inside the codebase* before changes (read-only) | Phase 1, before any non-trivial change |
| **Research Worker** | Investigate *outside the codebase* — library docs, specs, prior art (read-only) | Phase 1 (parallel to audit) or whenever a decision needs facts the codebase doesn't contain |
| **Feature Implementer** | Build NEW behavior → one commit | Phase 2, when the assignment adds new behavior |
| **Refactor Implementer** | Restructure WITHOUT changing behavior — DRY / consolidation / deepening | Phase 2, when the assignment is structural-only |
| **Fix Implementer** | Fix a broken behavior using TDD (failing test first, then green) | Phase 2, when the assignment is a bug / regression |
| **Functional Verifier** | *Does it work?* — builds, tests, scope, correctness, security | Phase 3 — always, parallel |
| **Architecture Verifier** | *Is it well-built?* — DRY, simplicity, N+1, deepening, dead code | Phase 3 — always, parallel |
| **Black-User E2E Validator** | *Does a clueless user succeed?* — drives the running app as a fresh user | Phase 3 — default parallel (skip only when trivial + no UI/flow impact) |
| **Design Verifier** | *Is it 1-tier quality?* — typography / color / spacing critique against reference tiers | Phase 3 — parallel when UI touched |

The non-negotiable pair on every commit is **Functional + Architecture** verifiers. They answer two distinct questions one verifier can't reliably do both of at adversarial depth.

**Two axes structure the persona library:**

- **Audit vs Research** (Phase 1): Audit reads *your* codebase; Research reads *the outside world*. Different tools (`grep`/`Read`/`SQL` vs `WebFetch`/`WebSearch`/Context7 MCP), different outputs (file:line + invariants vs facts + URLs). Both read-only, both parallel-safe.
- **Feature vs Refactor vs Fix** (Phase 2): the Implementer split by intent. Feature adds new behavior. Refactor preserves behavior + improves structure (Architecture Verifier is the main critic; all existing tests stay green with their existing assertions). Fix is TDD bug repair (Functional Verifier is the main critic; a failing reproducer test gates the commit, and the test must catch the inverted fix). Each has a distinct procedure, distinct mindset, and distinct PASS/FAIL bar.

## Structured handoff

Every worker — implementer, auditor, verifier — returns the same five fields:

1. **What was implemented** — concrete diffs, files touched, decisions made.
2. **What was left undone** — `BLOCKED` / `NEEDS_INFO` / `DONE_WITH_CONCERNS`.
3. **Commands run + exit codes** — the audit trail for "the work actually happened."
4. **Issues discovered** — severity-tagged (CRITICAL / HIGH / MID / LOW).
5. **Procedures followed** — which rules / memory / conventions were applied.

Without this, multi-agent work degenerates into "trust me, it worked."

## Dispatch description convention

Every `Agent` dispatch's `description` field starts with a role tag — so the Claude Code agent switcher shows the role at a glance instead of indistinguishable `general-purpose` lines:

```
[worker:feature] D2 extras 재설계 — table → card grid
[worker:refactor] settle engine — extract currency formatter
[worker:fix] calendar empty-month nav — wrong month resolves
[validator] functional — commit a3f2
[validator] architecture — commit a3f2
[validator] e2e — commit a3f2 (black user, add deal flow)
[validator] design — commit a3f2
[read] audit settlement amount touch-points
[research] react-aria menu keyboard nav conventions
```

Two subtype rules:
- `[worker:<intent>]` — intent (`feature` / `refactor` / `fix`) lives **inside** the bracket because each maps to a different persona prompt.
- `[validator] <verifier-type>` — verifier type lives as the **second word** because all four verifiers share the same procedural shell with different focus.

## When this skill fires

A single linear pass works fine for "fix this typo." This skill fires when:

- The task spans **multiple files or domains** (frontend + backend + DB, or design + functional + security).
- The user values **quality higher than speed** ("prod-ready", "1-tier", "real users will see this") and a single Claude session can miss things its own bias hides.
- The work needs **adversarial eyes** the orchestrator can't provide for itself (testing its own design, reviewing its own code).
- The user explicitly wants **autonomous progress** without micro-confirmation at each step.

## Model policy

Personas don't pin a model name — they pick a **tier by risk**. A dispatched agent inherits the session model by default (almost always right); you override only with a stated reason. Read-only mapping (Audit / Research) and routine work run on the efficient tier; verifiers escalate to the frontier tier when a commit carries money math, a security boundary, or a named 1-tier quality bar. Expressing model choice as *tier + reason* instead of a version name keeps the skill from silently aging every time the model line moves (4.6 → 4.8 → …). Full rationale in [`SKILL.md`](SKILL.md#model-policy).

## Install

Drop the skill into your local Claude Code skill directory:

```bash
git clone https://github.com/WonjunJoe/claude-orchestration-skill.git \
  ~/.claude/skills/orchestration
```

Claude Code auto-loads skills from `~/.claude/skills/` on session start. Verify by checking the available-skills list in any new session — `orchestration` should appear with the trigger phrases.

## Files

```
SKILL.md                              # Skill entry point (loaded by Claude Code on trigger)
references/
├── personas.md                       # Full prompt templates for all 9 agent personas
├── workflow.md                       # Phase-by-phase mechanics + decision trees
├── scrutiny-rules.md                 # Rule catalog read by Functional + Architecture Verifiers
├── ios-testing-conventions.md        # iOS-specific: accessibilityIdentifier + XCUITest
└── examples.md                       # Real production session walkthrough (4 rounds, 25 commits)
```

## Background

Extracted from a solo-built production SaaS shipping on FastAPI + Next.js + Supabase, where the orchestration loop drove 1,200+ commits across the codebase with adversarial verification at every step. The skill is the codified version of what worked.

## License

[MIT](LICENSE)
