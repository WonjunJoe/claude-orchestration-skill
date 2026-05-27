---
name: orchestration
description: Multi-agent orchestration loop for complex multi-step work. The main session acts as an orchestrator, delegating implementation to fresh-context worker agents and verification to independent adversarial validators, repeating verify→fix cycles until PASS. Use this skill whenever the user requests a complex multi-page redesign, refactoring across many files, design improvement with verification, or autonomous progress (e.g. "알아서 진행", "verify 도 같이", "ralph 처럼 돌려", "loop 으로", "PASS 까지"), or when a single task touches 3+ files OR requires both implementation and design review OR will produce 5+ commits. Also trigger when adversarial validation matters — security boundaries, settlement/financial accuracy, design quality at "1티어" tier (Stripe / Linear / Apple level), or when the user says they don't want to be asked at every branch. Even partial signals like "오케스트레이션", "워커 + 검증", or describing a workflow that needs independent eyes should bring this skill in.
---

# Orchestration — Multi-agent loop

A workflow for orchestrating complex tasks across specialized agents: workers implement, validators verify, verifiers critique with adversarial eyes. The main session is the **orchestrator** — it plans, dispatches, integrates, and decides when to ship.

This skill is not a script you run. It is a **mental model + persona library + workflow template** that the orchestrator (you, the main session) loads when a task is too big or risky for a single linear pass.

## When this fires (and why it works)

A single linear pass works fine for "fix this typo" or "rename this variable." It breaks down when:

- The task spans **multiple files or domains** (frontend + backend + DB, or design + functional + security).
- The user values **quality higher than speed** ("1티어", "prod-ready", "real users will see this") and a single Claude session — even a careful one — can miss things its own bias hides.
- The work needs **adversarial eyes** the orchestrator can't provide for itself (testing its own design, reviewing its own code).
- The user explicitly wants **autonomous progress** without micro-confirmation at each step.

The pattern works because each agent gets a **fresh context** and a **narrow remit**. The worker implementing a fix can't sneak its own design opinions past a validator who has never seen the code before. The orchestrator stays clean — it doesn't try to be implementer + critic + tester at once.

## Four phases

```
Phase 1 — Audit (parallel-safe, read-only)
   ↓ findings + fix proposals
Phase 2 — Implement (serial, fresh worker per feature)
   ↓ commits + 5-field handoffs
Phase 3 — Verify (independent context, adversarial)
   ↓ PASS / NEEDS_REVISION / FAIL
Phase 4 — Loop
   ↓ NEEDS_REVISION → critique-driven fix worker → back to Verify
   ↓ PASS → ship + report
```

### Phase 1 — Audit

Before any code changes, **dispatch read-only audit workers in parallel** to map the territory. Audits don't conflict with each other (read-only) so they go in one batch.

Three common audit types:

- **Domain Audit** — map current code + DB + business rules against the proposed change. Returns: where the change touches, what invariants matter, what edge cases lurk.
- **Codebase Audit** — find structural opportunities (deepening, DRY, dead code). Returns: prioritized refactor candidates with deletion-test reasoning.
- **Design Audit** — 1-tier designer eyes on the current screens. Returns: typography / spacing / color / hierarchy issues with reference patterns.

Skip audit when the task is small and the territory is well-known. Don't skip it when you don't already know the answer to "where in the codebase does this live and what does it touch?"

### Phase 2 — Implement

**Serial.** One worker at a time. Next worker inherits the previous commit via git. This avoids merge conflicts and keeps the architecture coherent.

The exception: workers in **completely separate file regions** (e.g., a frontend page and a backend endpoint that don't share files) can run in parallel — but only when you've verified there's no overlap. When in doubt, serial.

Each worker:
- Gets a **fresh context** (a clean Agent dispatch, not a continuation).
- Implements **one logical commit** worth of work. Big changes split into 2-4 commits.
- Returns a **5-field handoff** (see Structured handoff below).

### Phase 3 — Verify

**Independent context.** Each verifier has never seen the code. This is the whole point.

**Default = 3 verifiers in parallel** for every implementer commit:

1. **Functional Verifier** — *does it work?* Runs `npx tsc --noEmit` / `npm run build` / `pytest` / linters. Re-traces the diff against the original task. Checks edge cases, error paths, security boundaries. Reads the `[General] Functional correctness` + `[General] Surgical scope` + `[General] Layout shift` + stack-specific functional rules in `scrutiny-rules.md`.
2. **Architecture Verifier** — *is it well-built?* Critiques HOW the change is implemented: DRY, simplicity, deepening, N+1 / perf, dead code, premature abstraction, terminology consistency. Reads the `[General] Simplicity` + `[General] DRY violations` + `[General] Terminology consistency` + stack-specific architecture rules (`[ORM] N+1 absolute prohibition`, etc.) in `scrutiny-rules.md`.
3. **Black-User E2E Validator** — *does a clueless real user succeed?* Drives the actual running app via playwright / simctl. Logs in as a real user with **no prior knowledge of the diff or the codebase** — only knows the user-facing task ("add a deal", "view this month's settlement"). Looks for what only a fresh user would notice: confusing labels, dead-end flows, regressions in adjacent screens.

**Plus, conditional:**

4. **Design Verifier** — *is it 1-tier quality?* Senior designer persona, plays back screens, critiques typography / color / spacing / hierarchy against named reference tiers (Stripe / Linear / Apple). **Dispatch only when the commit touched UI.**

So the default parallel batch is **3 verifiers** (Functional + Architecture + Black-User E2E), or **4 verifiers** when UI changed (+ Design).

**Skip rule (skip e2e only):** when the change is tiny + obvious + has no UI / flow impact (e.g. a 1-3 line backend constant tweak, a string rename, a comment), drop Black-User E2E and run just **Functional + Architecture**. Never skip below 2. Functional + Architecture are non-negotiable on every commit.

**iOS / mobile stack: Functional Verifier 는 XCUITest 실행 의무.** Code trace 만으론 false positive 잦음 (예: `testBackPreservesSession` 가 PASS 단 실제 사용자는 lost — assertion 약함). XCUITest 가 진짜 end-user gesture verify. `xcodebuild test -only-testing:<UITestTarget>` 통과가 PASS 의 1st signal.

**Default model: Sonnet** for all verifiers. Fresh context + adversarial prompt is what makes them catch issues — not raw model power. Reserve Opus for implementers whose creative judgment is the bottleneck (e.g. visual design where Sonnet defaults have been rejected before).

**Main session does zero direct verification.** Don't read one screenshot and call it a critique; don't run a quick grep and call it scrutiny. Dispatch a fresh verifier instead. The whole point of the loop is that the orchestrator does not become a fifth opinion on its own work — it would carry the same blind spots.

### Phase 4 — Loop

Verifier returns one of three verdicts:

- **PASS** — ship. Report to user.
- **PASS_WITH_CONCERNS** — ship the main change. Concerns become follow-up tickets (backlog).
- **NEEDS_REVISION** — dispatch a fix worker with the verifier's critique handed in verbatim. Then re-verify.

**Round cap: 5.** If verification still fails after 5 rounds, the issue isn't implementation — the reference / direction is wrong. Stop and surface this to the user: "we've tried 5 rounds, the gap suggests we're aiming at the wrong target."

## Personas

This skill carries five reusable agent personas. Load the full persona prompt from `references/personas.md` when dispatching — that file is the source of truth for the system prompts.

| Persona | Role | Default model | Mandatory tool | Tools | When to dispatch |
|---|---|---|---|---|---|
| **Domain Audit Worker** | Map territory before changes | Sonnet | — | Read, Grep, SQL via MCP if available | Phase 1, before any non-trivial change |
| **Implementer Worker** | Build one feature → one commit | Sonnet (Opus when creative judgment dominates — visual design, copy with strong voice) | — | Read, Edit, Write, Bash | Phase 2, for any code change of 5+ lines or 2+ files |
| **Functional Verifier** | *Does it work?* — builds, tests, scope, correctness, security | Sonnet | `xcodebuild test -only-testing:<UITestTarget>` (iOS) | Read, Bash | Phase 3 — **always, default parallel** |
| **Architecture Verifier** | *Is it well-built?* — DRY, simplicity, perf (N+1), deepening, dead code | Sonnet | — | Read, Grep, Bash | Phase 3 — **always, default parallel** |
| **Black-User E2E Validator** | *Does a clueless user succeed?* — drives running app as fresh user | Sonnet | Playwright (web) / XCUITest (iOS) | playwright MCP, Bash | Phase 3 — **default parallel** (skip only if change is trivial + no UI/flow impact) |
| **Design Verifier** | *Is it 1-tier quality?* — typography / color / spacing critique | Sonnet | — | playwright MCP, Read | Phase 3 — **default parallel when UI touched** |

See `references/personas.md` for full prompt templates. See `references/workflow.md` for detailed phase-by-phase mechanics and decision trees.

## Dispatch description convention (every Agent call)

The `description` field on every `Agent` dispatch **must start with a role tag in square brackets** so the user can read the agent switcher at a glance and know what each running agent is doing.

| Tag | Use for | Examples of personas |
|---|---|---|
| `[worker]` | Mutates code / writes files / commits | Implementer Worker |
| `[validator]` | Adversarial verification of existing work (read-only, judges PASS/FAIL) | Functional Verifier, Architecture Verifier, Black-User E2E Validator, Design Verifier |
| `[read]` | Read-only mapping of the existing codebase / DB / docs | Domain Audit, Codebase Audit, Design Audit |
| `[research]` | Read-only investigation of *external* knowledge (library docs, web, library source) | Context7 lookup, library API research, prior-art search |

Format:

```
description: "[worker] D2 extras 재설계 — table → card grid"
description: "[validator] functional — D2 extras commit a3f2"
description: "[validator] architecture — D2 extras commit a3f2"
description: "[validator] e2e — D2 extras commit a3f2 (black user, add deal flow)"
description: "[validator] design — D2 extras commit a3f2"
description: "[read] audit settlement amount touch-points"
description: "[research] react-aria menu keyboard nav conventions"
```

For `[validator]`, name the verifier type as the second word — `functional` / `architecture` / `e2e` / `design`. Three or four `[validator] ...` lines tend to be running in parallel after each implementer commit; the second word is what lets the user tell them apart in the agent switcher.

Rules:
- The tag is the **first thing** in the description, before any task summary.
- One tag per dispatch. If a worker would need two (e.g. "research then implement"), split it into two dispatches.
- Lowercase, exactly one of the four tags above. No improvising (`[fix]`, `[design]`, `[qa]` etc. are not allowed — they fragment the convention).
- The rest of the description is still the 3-5 word task summary as before.

Why this exists: the agent switcher shows description verbatim. Without the tag, three lines of `general-purpose ...` all look the same and the user can't tell which is implementing vs critiquing vs just looking. With the tag, the role is the first token they read.

## Structured handoff (every worker uses this)

The whole loop is held together by a simple report format. Every worker — implementer, validator, verifier — returns these five fields:

1. **What was implemented** — concrete diffs, files touched, decisions made. (For validators: `N/A` — they don't change code.)
2. **What was left undone** — `BLOCKED` / `NEEDS_INFO` / `DONE_WITH_CONCERNS`. Be specific about what would unblock.
3. **Commands run + exit codes** — every shell/test command, with results. This is what the orchestrator audits to know the work actually happened.
4. **Issues discovered** — things that weren't part of the assignment but turned up. Severity-tagged (CRITICAL / HIGH / MID / LOW).
5. **Procedures followed** — which project rules / memory / conventions were applied, and where exceptions were taken.

This format is non-negotiable. Without it, multi-agent work degenerates into "trust me, it worked."

## Meta rules

- **Don't ask the user at every branch.** They opted into autonomy. Ask only when the answer is genuinely in their head (domain meaning, UX preference, business invariant). Technical trade-offs are yours.
- **Don't expose internal jargon to the user.** "Worker A failed M3 round 2 with H-2 critique" means nothing to them. Translate into "the design audit found three issues; I fixed two and re-tested."
- **Plan once, dispatch, then move.** Don't spend the user's tokens re-debating the plan inside the orchestrator. The plan was the audit. The dispatch is the commitment. Course-correct on verification failures, not on second thoughts.
- **HTML reports over walls of text** when the user is going to skim. A `tmp/*.html` you write at the end of a long session is much more useful to them than a 2000-token message they have to scroll through.
- **Project rules win** over this skill if they conflict. Read CLAUDE.md and any `feedback-*.md` memory files and let those override defaults here. This skill is general advice; the project knows itself best.

## Artifact location

Workers and verifiers produce a lot of throwaway: screenshots, HTML mockups, build logs, scratch scripts. **Never let them land at the repo root.** Every dispatched worker prompt must include explicit output paths.

| Artifact kind | Goes to |
|---|---|
| Screenshot / Playwright / simctl capture | `.playwright-mcp/<purpose>-<date>/` (or `/tmp/<project>-*.png` for fully ephemeral) |
| HTML mockup / design candidate | `tmp/designs/<version>/` |
| AI experiment / one-off script | `.dev/scratchpad/` |
| AI plan / decision memo | `.dev/plans/` |
| Build output / cache | `/tmp/<project>-build/`, `/tmp/<project>-*.log` |
| Production code | `src/` only |
| Business truth documents | `docs/` |

Root holds README / CLAUDE.md / CHANGELOG / config only. If a worker drops something at root anyway, the orchestrator moves it immediately AND amends the next prompt to that worker with the path rule (so the worker doesn't repeat).

The reason: a polluted root noise-floods `git status`, the IDE navigator, and — critically — the next AI session's first impression of what the repo even is.

## Triggering this skill yourself

The orchestrator (you) should load this skill when:

- The user describes a workflow that needs multiple specialized agents.
- A single task obviously needs both implementation and review by different eyes.
- You catch yourself about to do five Edit calls in a row in the main session — that's the moment to back up and dispatch workers instead.
- The user uses any of the trigger phrases: "알아서", "ㄱㄱ", "loop", "verify 도", "ralph 식", "1티어", "orchestration", "워커", or anything similar.

Once loaded, follow the four phases. Use the personas. Return clean reports. Done.

## Examples

`references/examples.md` has a worked example from a real production session that produced this skill — a 25+ commit cycle through 4 verifier rounds that took an artist-facing portal from "high-school project" to "prod-ready, 1-tier" quality. Read it when you need to see how the abstract phases map onto real decisions.
