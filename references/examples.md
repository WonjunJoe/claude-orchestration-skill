# Worked example — artist-portal redesign cycle

This is a real session from May 2026 — a production SaaS for a classical music management firm — that produced this skill. The user asked to redesign an artist-facing portal at "1-tier" quality (Stripe / Linear / Apple-level polish). The session ran for ~25 commits across 4 verifier rounds, hitting PASS on round 4. This example walks through what happened so the abstract phases above become concrete.

## The task

User input, paraphrased: "the artist portal feels like a high-school project. make it 1-tier Apple-level. for the rest, just go."

This is the kind of input the skill is designed for: open-ended quality target, multiple pages affected, user does not want to micromanage, autonomous progress expected.

## Sizing (what the orchestrator did before dispatching anything)

- **Touches**: 5 artist pages (dashboard, calendar, deals, settlements, settings), shared sidebar, modal patterns. Frontend only initially — but as it turned out, also backend (display_status derive, settlement engine refactor).
- **Risk**: design quality is the user's headline concern. Security/data isolation already passed in earlier rounds but watch for regression. Settlement math: must not break.
- **Expectation**: quality. Multiple verifier rounds expected.
- **Split**: the orchestrator decided on a 4-phase plan up front: light/serious redesign → verifier-round fixes → polish → ship.

The orchestrator did not write this plan to a file. It held it in head and dispatched the first phase.

## Round 1 — Light + serious redesign

Implementer Worker dispatched: "Rebuild the artist portal in light mode with serious typography (Stripe Dashboard reference). Replace the previous dark/Spotify attempt that failed. Use `.artist-portal` scoped tokens. Don't touch admin."

Worker returned commit `9a3550e` with 6 files modified. 5-field handoff was clean. Build passed.

**Functional Verifier** ran — found minor lint things, no blockers. **Design Verifier** dispatched with the 1-tier prompt and reference set (Stripe Dashboard + Linear + Carnegie Hall).

**Verifier verdict**: NEEDS_REVISION. 4 HIGH issues:

- H1: TEST environment banner is a wide amber stripe — looks like an alert, not a chip. Linear uses a 4px stripe; Vercel uses a corner ribbon.
- H2: KPI cards visually merge into the body — card boundary is too subtle (0.55px border on a #FAFAFA background)
- H3: H1 (the artist's display name) is 44px / weight 600 / -1.1px letter-spacing — that's marketing-hero scale, not product-dashboard scale. Stripe page H1s are 28px / 600 / -0.4px.
- H4: "입금 확인 중" is rendered as raw Korean text where the other KPI cells render numbers. Visual weight is asymmetric.

Plus 11 MID issues (sidebar logo, sidebar nav active state, avatar color, D-day chip, category bar overuse of color, etc.) and 7 LOW issues (polish).

## Round 2 — Fix worker dispatched with verifier critique verbatim

The orchestrator pasted the verifier's full `Issues discovered` section into the next Implementer Worker prompt. Did not paraphrase. The worker was told: "Address every HIGH issue. Address MID where the fix is clear. Defer LOW to a follow-up."

Worker returned commit `b9896c3` with 13 issues addressed:

- H1: amber stripe → small `[TEST]` chip in topbar corner, virtualToday as tabular-nums clock value.
- H2: body background `#FAFAFA → #F7F7F7` (slight darker), card border 0.55 → 1px, radius 12. Cards visually emerge from background.
- H3: H1 `44px / 600 / -1.1px → 28px / 600 / -0.4px`. Hero proportions corrected.
- H4: Added `kind="status"` variant to KpiCell so non-numeric values get a chip treatment, with a reserved-space `minHeight: 33` to prevent layout shift when data flips.
- M-series: sidebar accent bar on active nav, avatar grey + chevron icon, D-day amber chip with reserved width, category bar collapsed to grey + one accent, deals page sticky column header, settings input outline + focus ring, etc.

**Design Verifier** re-ran on `b9896c3`. Verdict: **NEEDS_REVISION (minor)**. 0 HIGH, 2 MID RESIDUAL:

- M5 RESIDUAL: calendar category chips are still multicolored. The worker had only removed the colored boxes around the chips; the dots inside remained colored. Design Verifier wanted full monochrome with a single accent for the active filter.
- M1 RESIDUAL: collapsed sidebar shows the product logo box and a chevron box floating as two disconnected rectangles. Worker had de-color-ed them but hadn't unified the frame.

Plus 5 new minor issues (calendar empty-month state, sub-meta information hierarchy, settlements summary too sparse, etc.).

## Round 3 — More targeted fix

Orchestrator dispatched another Implementer Worker, this time with both the Round 2 RESIDUAL set and the new N-issues bundled. Worker returned commit `d573ea6`:

- F1: calendar filter chip dots → all grey `var(--artist-fg-3)`. Calendar deal chip border-left also flipped to grey.
- F2: Sidebar default state changed from collapsed to expanded. New users now see the nav structure on first load.
- F3: Calendar empty-month state added with a single muted line ("이번 달 등록된 공연이 없습니다.").
- F4: Deals row sub-meta ("+1명 함께 출연") gets indent and an extra muted level.
- F5: Settlements summary card added at page top with rolling 12-month totals.

**Design Verifier** re-ran. Verdict: **NEEDS_REVISION** again, but only **one issue** this time.

- F1.A through F1.E: 5 specific dot/border-left sites verified at the DOM level to confirm they were really `rgb(82, 82, 91)`. They were not — the worker had only updated the filter chip and the calendar deal chip's main border, but had missed the smaller dots inside the deal chips and the modal dots.

Round 3 had improved most of the polish, but missed a literal handful of dot sites.

## Round 4 — Tiny fix

Orchestrator dispatched a small worker: "Change these 5 specific dot sites to `var(--artist-fg-3)`. Here are the file:line citations from the verifier."

Worker returned commit `158b70e`. 5 sites fixed, total diff: ~15 lines.

**Design Verifier** re-ran one last time. Sampled DOM computed styles on all 5 sites. Each one came back `rgb(82, 82, 91)`. Sampled adjacent sites for visual coherence.

**Verdict**: **PASS**. "Single-tone discipline applied uniformly. No new issues. prod-ready."

## What the loop produced

- 4 implementer commits + 4 design verifier runs + 2 scrutiny runs over ~2 hours of orchestration.
- A redesign that the user later described as "전체적으로 디자인 굉장히 많이 개선된듯. 좋다."
- Zero direct edits from the orchestrator after the first dispatch — it stayed clean and only made decisions.
- A pile of HTML reports the user could skim later (verifier critiques per round, design audit catalog, sample screenshots).

## What this example teaches

- **Audit first, even on a "just do it" task.** The Design Audit upstream of the implementer prevented the worker from guessing what "1-tier" meant.
- **Paste the verifier's words.** Each round of fix used the verifier's `Issues discovered` text verbatim. The implementer didn't have to interpret — it just executed.
- **The round count is normal.** PASS on round 1 would have been suspicious. Real quality work takes 2-4 rounds. The skill's 5-round cap is a safety net, not a target.
- **Different verifiers for different risks.** Scrutiny caught build issues. Design Verifier caught visual issues. They didn't overlap.
- **Small fixes get small dispatches.** The final round's worker had a 15-line job. The orchestrator didn't over-instrument it.

## What the orchestrator would do differently next time

- More aggressive parallel audits at the start. The Design Audit + Domain Audit could have run together.
- Earlier `tmp/*.html` report so the user could check in between rounds rather than at the end. This particular user opted into autonomy and was fine waiting, but other users would want sooner checkpoints.
- A pre-flight "is the implementer using the right reference?" check before round 1. The first attempt (a discarded Spotify-dark redesign that preceded this entire cycle) would have been caught at this gate.

The loop works, but the orchestrator can always tune it. That's why this skill is a template, not a script.
