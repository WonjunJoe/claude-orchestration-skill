# Workflow — phase-by-phase mechanics

This file expands the four-phase outline in `SKILL.md` into operational detail: how to decide what goes in which phase, how to handle parallel vs serial, what to do when things fail, and what to skip.

## Deciding the shape before you start

When a task lands, the orchestrator's first move is to size it. Not to start coding. Sizing answers four questions:

1. **What does this touch?** — Frontend? Backend? Database? Many or few files? If you don't know, that's your first audit assignment.
2. **What's risky?** — Money math (settlement accuracy), security boundaries (data isolation), visible quality (design tier), or invisible quality (refactor regression). Risk drives which validators you'll need.
3. **What does the user expect?** — Speed or quality? They'll usually tell you in tone: "ㄱㄱ" / "rough draft is fine" → speed; "1티어" / "prod-ready" / "verify 도" → quality. Quality means more verification rounds.
4. **Can it be split?** — Big tasks should become 3-5 commits, not one mega-commit. Decide the split before dispatching anything.

After this, sketch a one-paragraph plan — what audits, what implementer commits, what verifiers. You don't need to write it down formally; just hold it in head before dispatching.

## Phase 1 — Audit

### When to skip

Skip audit when:
- The task is small (1-3 files, you can see them all in your head).
- You've audited this exact area in the same session and nothing's changed.
- The user is asking for something simple and well-bounded ("rename this enum value across the codebase").

Don't skip when:
- The task crosses domains you don't have full models of (BE + FE + DB).
- The user used a vague directive ("clean up this page") — the audit IS the spec.
- You suspect existing bugs or inconsistencies the change might trip over.

### Dispatching audit + research in parallel

Read-only dispatches don't conflict. Send them all at once. **Audit Worker reads YOUR codebase; Research Worker reads the OUTSIDE world.** Both safe in one batch:

```
Dispatch 1: Domain Audit Worker — "map all places settlement amounts are computed/displayed, including BE response schemas and FE display paths"

Dispatch 2: Codebase Audit Worker — "find refactor opportunities in the artist portal that have surfaced in the last 5 commits; output prioritized list"

Dispatch 3: Design Audit Worker — "1-tier designer review of /my/calendar and /my/deals against Stripe/Linear references"

Dispatch 4: Research Worker — "react-aria v3.x — does MenuTrigger handle Escape-to-close natively, or do we need a handler? Cite the docs + version."
```

These return in parallel. The orchestrator integrates their findings before dispatching any implementer. Dispatches 1-3 all use the **Domain Audit Worker persona** with different task assignments (the "three audit types" — same persona prompt, task differs). Dispatch 4 uses the **Research Worker persona** (different tools, different sources, different output format).

### What an audit produces

A useful audit returns:

- **Where** — file paths, line numbers, function names. Not "somewhere in the artist code."
- **What** — the actual current behavior or current state, not a paraphrase.
- **Why it matters** — connection to the task, to a known rule, or to an invariant.
- **Fix proposal** — concrete enough that the next worker can act on it without thinking too much.

If an audit returns "everything looks fine and here are some ideas," dispatch was wasted. Push back into the audit's prompt for more specificity next time.

## Phase 2 — Implement

### Pick the right Implementer persona first

Three implementer personas, by **intent** (not by stack):

| Intent | Persona | Use when |
|---|---|---|
| Add NEW behavior | **Feature Implementer** | Assignment introduces something the system didn't have — new endpoint, new screen, new business rule, new external integration |
| Restructure WITHOUT changing behavior | **Refactor Implementer** | DRY extraction, consolidation, deepening, dead code removal, file reorganization, renames. All existing tests stay green with existing assertions |
| Fix a broken behavior | **Fix Implementer** | Bug report, failing test, regression, wrong calculation, security hole. TDD: failing reproducer test first, then green |

The persona choice changes the procedure, the catalog sections to apply, and **which verifier is the main critic**. Mis-picking is itself a bug — e.g. dispatching a Feature Implementer for a refactor task → they'll subtly change behavior because their persona doesn't enforce parity.

Self-check: if you can write the assignment as "before this commit X was Y; after this commit X is Z" where Y ≠ Z, it's Feature or Fix (which Z?). If "before and after are observable-equivalent," it's Refactor.

### Serial by default

The next worker should inherit the previous worker's commit via git. They start their context fresh, but they start with the codebase at HEAD.

```
Audit returns → Orchestrator picks first commit slice → Dispatch Implementer 1 (Feature/Refactor/Fix)
  → Implementer 1 commits SHA X → Orchestrator dispatches Validators on SHA X
  → Validators pass → Orchestrator dispatches Implementer 2 on commit slice 2
  → ... and so on
```

This serial chain is slower than parallel but keeps coherence: no merge conflicts, no two workers reaching different conclusions about the same abstraction, no surprise interactions.

### Parallel exceptions

Parallel implementation is OK when **and only when**:

- The two workers touch **disjoint file regions** (one in `frontend/`, one in `backend/`, with no shared utility being edited).
- Neither change requires knowing the other's outcome.
- The user is OK with a possible two-commit ordering inversion (parallel commits land in race order).

When in doubt, serial. The cost of redoing work after a merge conflict is much higher than the cost of waiting an extra few minutes.

### Splitting big changes

If a worker would produce 200+ lines of diff or touch 5+ files, split into 2-4 commits. Each commit should have **one reason to exist** ("extract the helper", "use the helper in callers", "delete the inline version"). This makes review much cheaper and rollback much safer.

Suggest the split in the worker's prompt — don't leave it to the worker to figure out commit shape.

## Phase 3 — Verify

### Which verifiers for which changes

**Default: dispatch Functional + Architecture + Black-User E2E in parallel** for every implementer commit. Add **Design Verifier** in parallel when the commit touched UI. Sonnet × 3-4 is cheap compared to a single orchestrator miss that the user catches in dogfood and forces a full round-back.

**The non-negotiable pair: Functional + Architecture.** Every commit gets both, no exceptions. These two answer the two distinct questions every change must pass — *does it work?* and *is it well-built?* — and one verifier in one context can't reliably do both at adversarial depth.

**Black-User E2E is the default third, skip rule is narrow:** drop e2e only when the change is **trivial AND has zero UI / flow impact** — a 1-3 line backend constant tweak, a comment, an internal helper rename that's not user-facing. The moment a change touches anything a user can see or interact with, e2e runs.

**Design Verifier is conditional**: dispatched in parallel with the other 3 only when the commit produced visible UI output (component change, page redesign, new screen, style change). Skip on pure backend / refactor / config commits.

**iOS / mobile stack: Functional Verifier 의 XCUITest 결과 = primary signal. Code trace 는 보조.** `xcodebuild test` exit 0 + XCUITest assertions all green 이어야 PASS 선언 가능. Black-User E2E uses simctl gesture for the user-flow side.

| Change type | Functional | Architecture | Black-User E2E | Design |
|---|---|---|---|---|
| Pure refactor (no behavior change) | ✓ | ✓ | optional | — |
| Backend logic (math, business rules) | ✓ | ✓ | ✓ | — |
| Backend security / permissions | ✓ | ✓ | ✓ | — |
| Frontend functional change | ✓ | ✓ | ✓ | ✓ if any visual |
| Frontend visual redesign | ✓ | ✓ | ✓ | ✓ |
| End-to-end feature | ✓ | ✓ | ✓ | ✓ |
| **Trivial — 1-3 line const / comment / rename, no UI** | ✓ | ✓ | — | — |
| Documentation / comments only | — | — | — | — |
| **iOS SwiftUI change (any)** | **✓ XCUITest mandatory** | **✓** | **✓ simctl gesture** | **✓ if visual** |

When in doubt, dispatch all three (or four with Design). Sonnet cost is far below "user catches it in dogfood and round 4 starts over."

### Reading verifier reports

The verifier's verdict is one of four:

- **PASS** — ship it. Mark the milestone done in the orchestrator's plan, move to next.
- **PASS_WITH_CONCERNS** — the main change is fine; the concerns are real but they're not blockers. Add them to a backlog file or mention them in the user-facing report. Don't loop on them in this cycle.
- **NEEDS_REVISION** — concrete issues, must be addressed before ship. Spawn a fix worker with the verifier's critique pasted in verbatim.
- **FAIL** — fundamental problem (wrong direction, broken approach, missing assumption). Don't just dispatch another fix worker. Stop and rethink. Often surface this to the user.

### Fix-worker handoff

When dispatching a fix worker after NEEDS_REVISION:

- Paste the verifier's `Issues discovered` directly into the fix worker's prompt. Don't paraphrase. The verifier's exact wording is what the worker needs to act on.
- Tell the worker which severity to address. Usually HIGH and MID; LOW often gets deferred.
- Be explicit: "After your fix, this same verifier will re-check. Address every HIGH issue, address MID issues unless you have a documented reason not to."

## Phase 4 — Loop

### The round cap

Five rounds. After that, stop and surface the gap to the user.

Why: if a 1-tier designer keeps finding issues after five attempts, the implementer isn't the problem — the reference is wrong, the constraints are wrong, or the goal was misread. Throwing a sixth implementer at it just burns tokens.

In practice, well-scoped tasks PASS in 1-3 rounds. NEEDS_REVISION on round 1 is normal. NEEDS_REVISION on round 4 is a signal.

### When to escalate to the user mid-loop

- Verifier reports something that wasn't in the original ask and looks load-bearing ("the design is fine but this feature appears to leak data between artists" — that's bigger than the assignment).
- A worker BLOCKED on `NEEDS_INFO` that only the user knows (domain meaning, business rule, UX preference).
- The same issue keeps reappearing across rounds — the worker is misunderstanding something the user could clarify in one sentence.

Default is to not interrupt. Interrupt only when interrupting saves the user time.

## Phase 4.5 — Ship

When the loop terminates with PASS:

1. **Verify the working tree is clean.** No uncommitted scratch files. Run `git status`.
2. **Push if the project's policy says push.** Read CLAUDE.md or memory files for the project's push policy — many projects auto-push to test branches but require explicit approval for main.
3. **Deploy if BE changes need it.** Match the project's deploy command (e.g., `fly deploy --config fly.test.toml`).
4. **Report to the user.** Concise summary — what changed, what verifier rounds happened, what's on the backlog, what's the next decision point. HTML report for anything beyond a few commits.

The report is the user's main artifact. Treat it as important as the code.

## Artifact location rule

Every worker and verifier dispatch must include explicit output paths in the prompt. Otherwise workers default to cwd (often the repo root) and pollute it with screenshots, build logs, and scratch files.

| Artifact kind | Path |
|---|---|
| Playwright / simctl screenshot | `.playwright-mcp/<purpose>-<date>/` |
| HTML mockup / design candidate | `tmp/designs/<version>/` |
| Build log / cache | `/tmp/<project>-build/`, `/tmp/<project>-*.log` |
| AI experiment / one-off script | `.dev/scratchpad/` |
| AI plan / decision memo | `.dev/plans/` |
| Production code | `src/` only |
| Business truth documents | `docs/` only |

Root holds README / CLAUDE.md / CHANGELOG / config only. If a worker still drops a file at root, the orchestrator moves it on receipt and amends the next prompt to that worker with the path rule. Save the rule as a memory feedback if the project lacks one — future sessions will inherit.

## What you don't dispatch

Some tasks shouldn't go through this loop at all. Direct implementation in the orchestrator is fine for:

- Single-line edits.
- Renaming a variable across a few files (often faster to do directly than to dispatch).
- Writing a config file from scratch.
- Reading and responding to a question.

The loop exists to manage risk and parallelism. If neither applies, just do the work.

## Token economy

Every dispatch costs context (the worker reads a fresh prompt + files), time (workers run sequentially), and tokens (the worker's whole session counts).

So:

- Don't dispatch for trivia.
- Don't over-spec the worker prompt. Workers are smart; one good paragraph beats five.
- Cache work: if a previous audit found the territory, the next implementer doesn't need their own audit.
- Re-use validators: a Functional Verifier that just verified commit X can verify commit X+1 in the same dispatch if X+1 is small.

The point is to minimize wasted work, not to maximize dispatches.

---

## iOS Testing Conventions

This section applies to any project using Swift / SwiftUI / Xcode. Reference when writing implementer prompts or Functional Verifier prompts for iOS changes.

Full detail: `references/ios-testing-conventions.md`.

### accessibilityIdentifier naming

Convention: `<screen>.<element>` — all lowercase, hyphen-separated.

| Screen | Element | Identifier |
|---|---|---|
| Today (Daily Plan Card) | Primary CTA button | `today.primary-cta` |
| Today | Rest day CTA | `today.rest-cta` |
| Logger | kg field for set j of exercise i | `logger.set.{i}.{j}.kg` |
| Logger | reps field for set j of exercise i | `logger.set.{i}.{j}.reps` |
| Logger | Add set button for exercise i | `logger.exercise.{i}.add-set` |
| Logger | Complete set button for set j of exercise i | `logger.set.{i}.{j}.complete` |
| Readiness | Self-report slider | `readiness.slider` |
| Readiness | Submit button | `readiness.submit` |
| TabBar | Any tab item | `tab.<tabname>` |

Rule: every `Button`, `TextField`, `.onTapGesture` interactive, and `TabView` tab item added by an implementer must have `.accessibilityIdentifier(...)`. No exceptions.

### Test target setup (project.yml / Xcode)

```yaml
# project.yml (XcodeGen) — add alongside main target
targets:
  WorkinUITests:
    type: bundle.ui-testing
    platform: iOS
    deploymentTarget: "17.0"
    sources: [WorkinUITests]
    settings:
      TEST_HOST: ""
    dependencies:
      - target: Workin
```

Or via Xcode: File → New → Target → UI Testing Bundle, name `WorkinUITests`.

### launchEnvironment — fresh state for UI tests

In `WorkinUITestsBase.swift`:

```swift
import XCTest

class WorkinUITestsBase: XCTestCase {
    var app: XCUIApplication!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchEnvironment["WORKIN_UI_TEST"] = "1"   // skips onboarding + auth sheet
        app.launchEnvironment["WORKIN_RESET_STATE"] = "1" // wipes local DB to fresh
        app.launch()
    }
}
```

In `AppDelegate` / `@main` App struct (DEBUG only):

```swift
#if DEBUG
if ProcessInfo.processInfo.environment["WORKIN_UI_TEST"] == "1" {
    // inject mock session, skip Apple Sign-in sheet
    // wipe SQLite DB if WORKIN_RESET_STATE == "1"
}
#endif
```

### Common XCUITest query patterns

```swift
// Tap primary CTA on Today screen
app.buttons["today.primary-cta"].tap()

// Type into kg field (set 0, exercise 0)
let kgField = app.textFields["logger.set.0.0.kg"]
kgField.tap()
kgField.typeText("100")

// Wait for element to appear (async data loads)
let card = app.staticTexts["today.plan-title"]
XCTAssertTrue(card.waitForExistence(timeout: 5))

// TabBar navigation
app.buttons["tab.history"].tap()

// Descendants inside a ScrollView / LazyVGrid
let grid = app.scrollViews.firstMatch
let cell = grid.buttons["today.exercise.0"]
cell.tap()
```

### Running XCUITest in verifier dispatch

Include in every Functional Verifier prompt for iOS changes:

```
After build verification, run XCUITest:
xcodebuild test \
  -project src/Workin.xcodeproj \
  -scheme Workin \
  -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
  -only-testing:WorkinUITests \
  2>&1 | tail -100

Report: exit code, number of tests run, any failures with file:line.
XCUITest PASS = primary PASS signal. Build PASS alone is insufficient.
```
