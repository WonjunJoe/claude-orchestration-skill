# iOS Testing Conventions

Reference for orchestration skill — applies to all Swift / SwiftUI / Xcode projects.
Load when writing **Implementer Worker** prompts (for the accessibilityIdentifier rule) and **Functional Verifier** prompts (for XCUITest mandatory step).

---

## 1. accessibilityIdentifier naming convention

**Every tap target added by an implementer must have `.accessibilityIdentifier(...)`.**

Naming pattern: `<screen>.<element>` — lowercase, hyphen-separated words, dynamic indices use `{n}`.

| Screen | Element | Identifier |
|---|---|---|
| Today (Daily Plan Card) | Primary CTA button | `today.primary-cta` |
| Today | Rest day CTA | `today.rest-cta` |
| Today | Plan exercise cell (index i) | `today.exercise.{i}` |
| Logger | kg TextField — exercise i, set j | `logger.set.{i}.{j}.kg` |
| Logger | reps TextField — exercise i, set j | `logger.set.{i}.{j}.reps` |
| Logger | RPE TextField — exercise i, set j | `logger.set.{i}.{j}.rpe` |
| Logger | Complete set button — exercise i, set j | `logger.set.{i}.{j}.complete` |
| Logger | Add set button — exercise i | `logger.exercise.{i}.add-set` |
| Readiness | Self-report slider | `readiness.slider` |
| Readiness | Submit button | `readiness.submit` |
| Readiness | Soreness tag chip (tag name) | `readiness.tag.{name}` |
| TabBar | Tab item | `tab.{tabname}` (e.g. `tab.today`, `tab.history`) |
| Onboarding | Next/Continue button | `onboarding.continue` |
| History | Session cell (index i) | `history.session.{i}` |

New screens: extend the pattern — don't invent a new scheme.

**SwiftUI snippet:**

```swift
Button("오늘의 운동 시작") { ... }
    .accessibilityIdentifier("today.primary-cta")

TextField("kg", text: $kg)
    .accessibilityIdentifier("logger.set.\(exIdx).\(setIdx).kg")
```

**Implementer grep before commit:**

```bash
# Find Swift files that introduced Button/TextField without accessibilityIdentifier
git diff --name-only HEAD | grep "\.swift$" | while read f; do
  if grep -q "Button\|TextField\|\.onTapGesture" "$f" && \
     ! grep -q "accessibilityIdentifier" "$f"; then
    echo "MISSING accessibilityIdentifier: $f"
  fi
done
```

---

## 2. Test target setup

### Via XcodeGen (`project.yml`)

```yaml
targets:
  WorkinUITests:
    type: bundle.ui-testing
    platform: iOS
    deploymentTarget: "17.0"
    sources:
      - path: WorkinUITests
    settings:
      TEST_HOST: ""
      BUNDLE_LOADER: ""
    dependencies:
      - target: Workin
```

### Via Xcode GUI

File → New → Target → UI Testing Bundle → name `WorkinUITests` → language Swift → Finish.

---

## 3. Fresh-state launchEnvironment pattern

Prevents test pollution from prior sessions. Required for deterministic UI tests.

**`WorkinUITestsBase.swift`** (all UI test cases inherit from this):

```swift
import XCTest

class WorkinUITestsBase: XCTestCase {
    var app: XCUIApplication!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        // Skip Apple Sign-in sheet; inject mock session
        app.launchEnvironment["WORKIN_UI_TEST"] = "1"
        // Wipe local SQLite DB to factory-fresh state
        app.launchEnvironment["WORKIN_RESET_STATE"] = "1"
        app.launch()
    }
}
```

**App entry point (DEBUG guard):**

```swift
// In @main App struct or AppDelegate, before any scene setup
#if DEBUG
let env = ProcessInfo.processInfo.environment
if env["WORKIN_UI_TEST"] == "1" {
    // Bypass onboarding, inject anonymous/mock session
    AppBootstrap.injectMockSession()
}
if env["WORKIN_RESET_STATE"] == "1" {
    AppBootstrap.wipeSQLiteDB()
}
#endif
```

`AppBootstrap` lives in `src/Debug/` — never ships in Release.

---

## 4. Common XCUITest query patterns

```swift
// ── Tap primary CTA ──────────────────────────────────────
app.buttons["today.primary-cta"].tap()

// ── Type into a TextField ────────────────────────────────
let kgField = app.textFields["logger.set.0.0.kg"]
kgField.tap()
kgField.typeText("100")

// ── Wait for async data (network / DB) ───────────────────
let title = app.staticTexts["today.plan-title"]
XCTAssertTrue(title.waitForExistence(timeout: 5), "Plan title did not appear")

// ── TabBar navigation ────────────────────────────────────
app.buttons["tab.history"].tap()

// ── Descendants inside LazyVGrid / ScrollView ────────────
let grid = app.scrollViews.firstMatch
let cell = grid.buttons["today.exercise.0"]
XCTAssertTrue(cell.waitForExistence(timeout: 3))
cell.tap()

// ── Assert element does NOT exist ────────────────────────
XCTAssertFalse(app.alerts.firstMatch.exists)

// ── Type then dismiss keyboard ───────────────────────────
kgField.typeText("80")
app.keyboards.buttons["Done"].tap()
// or:
kgField.typeText("\n")
```

---

## 5. Synchronous vs async

| Pattern | Use when |
|---|---|
| `element.exists` | Immediately after an action that should be synchronous |
| `element.waitForExistence(timeout: N)` | Async: DB fetch, LLM response, network call, animation |
| `XCTAssertTrue(pred.evaluate(with: nil))` | Complex NSPredicate conditions |

Default timeout: **5 seconds** for data loads, **2 seconds** for UI transitions.

---

## 6. Running XCUITest — command for Functional Verifier dispatch

Include verbatim in the Functional Verifier prompt for any iOS change:

```
iOS XCUITest (mandatory before PASS verdict):

xcodebuild test \
  -project src/Workin.xcodeproj \
  -scheme Workin \
  -destination 'platform=iOS Simulator,name=iPhone 15 Pro' \
  -only-testing:WorkinUITests \
  2>&1 | tail -120

Report:
- Exit code (0 = all pass, non-zero = failures)
- Total tests run + failures
- Any "element not found" or assertion failure with file:line
- If test count < expected (e.g., 0 tests ran): build-for-testing issue — flag as HIGH

XCUITest exit 0 + all assertions green = primary PASS signal.
`xcodebuild build` exit 0 alone is insufficient — code compiles but users can still be broken.
```

---

## 7. Assertion strength check (Functional Verifier responsibility)

Weak assertions pass even when the feature is broken. The Functional Verifier must spot these.

**Weak (flag as MID):**
```swift
XCTAssertTrue(app.buttons["today.primary-cta"].exists)
// This passes even if the button is off-screen or disabled
```

**Strong:**
```swift
let cta = app.buttons["today.primary-cta"]
XCTAssertTrue(cta.waitForExistence(timeout: 3))
XCTAssertTrue(cta.isEnabled)
XCTAssertTrue(cta.isHittable)  // not obscured by another element
```

Mentally invert the fix — would the test catch the regression? If not, flag it.
