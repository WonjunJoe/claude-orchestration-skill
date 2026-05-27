# Rule Catalog (Functional + Architecture Verifiers)

This file is the source of truth for technical rule checks. It is read by **two distinct verifiers** in Phase 3, each applying a different subset:

- **Functional Verifier** (*does it work?*) reads: `Functional correctness`, `Surgical scope`, `Layout shift`, plus the functional/security parts of stack sections (build commands, route invariants, XCUITest mandatory step, security smells).
- **Architecture Verifier** (*is it well-built?*) reads: `Simplicity`, `DRY violations`, `Terminology consistency`, plus stack-specific architecture rules (`[ORM] N+1 absolute prohibition`, component-reuse signals, router structure, internal-module-mock TDD violations, etc.).

The catalog itself doesn't change between verifiers — only which sections they apply. See `personas.md` for each verifier's exact procedure.

The catalog covers: **기능 검증 (functional) → 정합성 (correctness) → 효율성 (efficiency) → 보안 (security) → 일관성 (consistency)**.

## Stack auto-detection

Read these files (existence checks) before applying rules:
- `package.json` with `"next"` dep → apply **[Web — Next.js]**
- `package.json` with `"react"` (no `"next"`) → apply **[Web — React]**
- `pyproject.toml` / `requirements.txt` with FastAPI → apply **[API — FastAPI]**
- `pyproject.toml` / `requirements.txt` with SQLModel/SQLAlchemy/Prisma → apply **[ORM]**
- `*.xcodeproj` or `Package.swift` → apply **[iOS — SwiftUI]**

Always apply **[General]**. Stack sections add to general, not replace.

---

## [General] — All projects

### Functional correctness (기능 검증)
- Diff implements the stated task? Re-read the user request and trace it to the commit
- Edge cases handled? null/empty/0/단일 element/max-size — at least one of each tested
- Error paths exercised? Tests should fail when the fix is mentally inverted ("fail-first" reasoning)
- New code path actually reachable? Dead branch / unreachable handler 위반

### Surgical scope (의도 일관성)
- Diff has changes to files/lines unrelated to the stated task → scope creep 위반
- Reformatting of adjacent untouched code → 위반 ("건드린 것만 건드려라")
- "Cleanup" of dead code that user didn't ask to remove → 위반
- New abstractions/helpers not justified by the task → over-engineering

### Simplicity (효율성 — 코드)
- if/else 3단 이상 중첩 → 거의 항상 잘못된 추상화
- 함수 시그니처 5+ 파라미터 → 책임 분리 필요
- 응답 schema 10+ 필드 → 진짜 다 필요한가
- "혹시 모르니까" 류 방어 코드 (내부 호출 결과 재검증, 일어날 수 없는 시나리오 에러 핸들링) → 위반
- 새 추상화가 같은 패턴 3번 등장 전 도입 → premature abstraction

### DRY violations (중복 제거)
Grep the diff:
- Inline hex color literals (`#[0-9a-fA-F]{3,8}`) when design token system exists → use tokens
- Inline formatters (`toLocaleString`, `.toFixed`, `Number(...) + "원"`) appearing 2+ times → extract helper
- Duplicated UI structure (button styles, table layouts) in 2+ files → extract component
- 같은 비즈니스 룰이 BE/FE 양쪽 inline → 한쪽이 source of truth, 다른쪽 import 또는 enum 공유

### Terminology consistency
- If project has `CONTEXT.md` "통일 용어 사전" or equivalent glossary, grep diff for forbidden variants
- JSX text / placeholders / chart labels / table headers / dialog titles 모두 dictionary 와 매치
- 같은 도메인 개념은 같은 단어 (예: 다 "정산 기준 금액", 일부 "총액" 금지)
- 변수명 (영어 OK) 과 사용자 라벨 (한국어, 사전) 구분되어 있는지

### Layout shift (UI-bearing projects)
- Variable-width numeric/text cells without `minWidth` + `tabular-nums` → layout shift 위반
- Active tab states changing font-weight/padding (옆 요소 1-3px 밀림) → 위반
- State-dependent button labels ("확인" ↔ "확인 취소") without minWidth on button → 위반
- Modal/inline form toggle that pushes list below → use inline draft row 대신
- 시뮬레이션: "값 바뀌면 옆 요소가 픽셀 단위로 움직이나?" Yes = FAIL

### Empty success notifications
- Grep diff for `alert(`, `toast.success(`, `dialog.show(` on success path (try 블록 끝, catch 밖) → 위반
- Tutorial/usage text in UI ("여기를 클릭하세요", "셀 클릭으로 수정") → 위반 (UI 가 자명해야)
- Wizard tone ("여기서는 X 를 할 수 있습니다") → 위반
- 단 placeholder, button label, title 속성, 빈 상태 메시지 ("등록된 데이터 없음") 는 OK

### Influence radius (영향 범위)
- POST/PATCH/DELETE without corresponding cache mutate calls (SWR / React Query / Zustand) → stale UI 위험
- New BE endpoint without FE updates that consume it (orphan endpoint)
- Mutation invalidating only direct cache, not transitive (예: extras 추가 시 dashboard cache 도 invalidate)

### Security (보안)
- Hardcoded secrets in diff (API key, password, token format) → **BLOCK** (CRITICAL)
- `.env` / credentials file in staged files → **BLOCK** (CRITICAL)
- SQL via string interpolation (f-string, template literal) → injection risk (CRITICAL)
- `dangerouslySetInnerHTML` / `eval` / `innerHTML` on user input → XSS (CRITICAL)
- Auth check missing on new router/endpoint → privilege escalation (HIGH)
- 응답에 sensitive 필드 (password_hash, internal_id, 다른 사용자 데이터) → leak (HIGH)
- CORS / CSRF setup missing or `*` wildcard → (MID)

### TDD discipline
- `mocker.patch("우리.모듈.X")` / `vi.mock("@/lib/api")` / monkeypatch on our own functions → 내부 mock 금지
- Assertions on call count / call order on internal methods → 구현 결합
- One test asserting 5+ unrelated things → split per behavior
- Tests named after class/method/file (`test_FooClass_bar_method`) → 구현 결합 signal
- Private method (`_foo`) 직접 호출/검증 → 위반

### Knowledge capture
- 비자명한 디버깅 끝났는데 `.dev/troubleshooting.md` 변동 0 → 위반
- 재사용 가능한 패턴 발견했는데 `.dev/learnings.md` 변동 0 → 위반
- 사용자 관점 마일스톤인데 `CHANGELOG.md` 변동 0 → 위반

---

## [Web — Next.js]

### Route group path collision
- New page in `app/(groupA)/X/page.tsx` when `app/(groupB)/X/page.tsx` exists → build fail "two parallel pages that resolve to the same path"
- `tsc` 통과해도 `npm run build` 만 실패 → push 전 build 한 번 필수
- Fix: introduce URL prefix per group (예: `(artist)/my/X/page.tsx`)

### Server / Client component boundary
- `'use client'` 없이 `useState`/`useEffect` 사용 → build error
- Server Component 에서 client 전용 API (`window`, `localStorage`) 접근 → runtime error
- 큰 client component 안에 작은 server fetch 부분 → SSR 손해, 분리 검토

### Table conventions (data display)
- Multi-row × multi-column structured data → `<table>`, not card list of inline text (정렬·비교 불가)
- Column headers should be sortable (`useTableSort` or equivalent hook)
- Row click → edit modal (not separate "Edit" button per row)
- Variable-width number cells → `tabular-nums` + `minWidth`

### Form / input
- `<input type="number">` spinner → globals.css override (`-webkit-appearance: none; -moz-appearance: textfield`)
- Number inputs should use `tabular-nums`

### Modal consistency
- All modals in app should share base structure (header / form-table / footer). 한 시스템 한 톤
- 개별 modal 의 visual identity (보라색 accent bar, 큰 chip 등) → 위반

---

## [Web — React (no Next)]

### General React conventions
- `useEffect` with empty deps but using props/state → stale closure risk
- Direct DOM manipulation (`document.getElementById`) → should be ref
- Missing `key` prop on list items
- Inline function as event handler in tight loop → re-render concern (단 micro-optimization 일 수 있음, 의도 확인)

### State management
- Same state in 2+ places (component state + global store + URL) → single source of truth 위반
- Prop drilling 4+ levels → context / store 도입 검토

---

## [API — FastAPI]

### APIRouter path
- `@router.get("/")` / post/put/delete → should be `@router.get("")` (avoid 307 redirect under reverse proxy)
- Detail endpoints `@router.get("/{id}")` 는 OK (`/` 가 path 구분자)
- FE 호출도 trailing slash 없이 — `/api/foo` (아니라 `/api/foo/`)

### Request/response
- Pydantic model with required fields and no default → ValidationError on partial body. 의도?
- Endpoint returning full DB row when summary suffices → response bloat
- Missing `response_model=` annotation → no validation on output
- 클라이언트가 안 쓰는 필드 반환 → API surface bloat

### Authentication
- Endpoint missing `Depends(get_current_user)` or equivalent → public exposure (HIGH)
- Role check in router body instead of dependency → harder to audit, easy to miss
- 응답에 password_hash, JWT secret, internal admin field 반환 → leak (CRITICAL)

### Async/sync
- `async def` endpoint with sync I/O (blocking `requests.get` etc.) → event loop block
- `time.sleep` in async context → 위반 (use `asyncio.sleep`)

---

## [ORM] — SQLModel / SQLAlchemy / Prisma

### N+1 absolute prohibition
- Loop accessing `obj.relationship` (lazy load) → N queries per iteration
- Loop with `session.exec(select(X).where(X.id == foo.x_id))` → N+1
- Fix: `selectinload` / `joinedload` in initial query, or batched IN query (collect IDs, one query, group by deal_id)

### Query budget (자기 검증)
- GET single resource: ≤ 5 queries
- GET list (with aggregates): ≤ 8 queries
- Heavy aggregate (dashboard): ≤ 12 queries
- Over budget → almost certainly N+1. 측정 방법: app_log query_count, request logger middleware, 또는 SQLAlchemy echo

### SQL JOIN guidance
- Repeated SELECT in loop → consider JOIN (반복문 단건 조회면 거의 항상 JOIN 후보)
- Explicit `ON` clauses (no NATURAL JOIN, no USING in production code)
- Alias all tables when 2+, prefix all columns
- `SELECT *` 금지 — 필요한 컬럼만
- `LEFT JOIN` 의 오른쪽 조건은 ON 절에 (WHERE 에 두면 사실상 INNER)
- 6+ 테이블 거대 JOIN → CTE (`WITH`) 단계 분리 검토
- 단순 존재 확인은 `EXISTS` / `IN` 이 더 효율적
- DISTINCT 로 JOIN 중복 덮어쓰기 → 조인 관계 자체 재검토

### Enum vs magic numbers
- Status/step comparisons should use IntEnum, not raw 0/1/2/3
- 매직 숫자 직접 비교 (`if step >= 3:`) → 위반, IntEnum 사용

### Transaction boundaries
- Multiple writes without `session.begin()` / explicit transaction → partial failure 위험
- Long transaction holding locks → contention

---

## [iOS — SwiftUI]

### Design tokens
- Hardcoded colors (`.red`, `.blue`, `Color(red:...)`) → use Color asset (`Color("Accent")`)
- Hardcoded fonts (`Font.system(size: 14)`) → use semantic style (`.font(.body)`) for Dynamic Type
- Hardcoded spacing (`.padding(16)` everywhere) → use spacing constants

### Dynamic Type support
- Fixed `.frame(height: ...)` on text-bearing views → may clip with Larger Text
- `Text` without `.lineLimit` + `.minimumScaleFactor` → may truncate ugly
- Custom font without `relativeTo:` → Dynamic Type 안 따라옴

### HealthKit / data (해당 시)
- `HKHealthStore.requestAuthorization` 매번 호출 → 1회만 (이미 grant 된 type re-request X)
- Background delivery setup needs `enableBackgroundDelivery` after authorization
- Privacy strings (`NSHealthShareUsageDescription` 등) in Info.plist — 누락 시 crash

### Concurrency
- `@MainActor` 안 붙은 UI 코드 in Task → main thread violation
- `async let` 안 await 함 → unstructured concurrency
- `Task { @MainActor in ... }` 패턴 적절히 사용?

### Accessibility
- Tappable view without `.accessibilityLabel` → VoiceOver 사용자 막힘
- Image without alternative text → 위반
- Custom controls without `.accessibilityIdentifier` → UI test 어려움

---

## Verdict format

After applying all relevant sections, return:

- **PASS** — no violations found
- **PASS_WITH_CONCERNS** — only MID/LOW violations, list as backlog
- **NEEDS_REVISION** — HIGH violations or 3+ MID — fix loop required
- **FAIL** — CRITICAL violation → **BLOCK** ship, fix mandatory

For each issue:
- **Severity**: CRITICAL / HIGH / MID / LOW
- **File:line**
- **Rule violated** (section name from this file, 예: `[General] DRY violations`)
- **Suggested fix** (concrete, not "improve this"; cite reference pattern if applicable)

Include this header in the report so the orchestrator can parse:

```
SCRUTINY VERDICT: <PASS / PASS_WITH_CONCERNS / NEEDS_REVISION / FAIL>
STACK DETECTED: <General + Next.js + ORM + ...>
ISSUES: <count by severity, e.g., "CRITICAL: 0, HIGH: 1, MID: 2, LOW: 4">
```

Then the issue list, then the 5-field handoff (per `personas.md`).
