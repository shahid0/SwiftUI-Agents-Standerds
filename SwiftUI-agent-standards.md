# SwiftUI iOS 17+ Application Development Standard (Draft)

**Document ID:** SUI-17-STD
**Version:** 0.4
**Status:** Draft
**Applies to:** iOS apps with minimum deployment target **iOS 17.0**
**Owner:** Engineering (iOS)
**Last updated:** 2026-01-14

## Change log
- **0.3 → 0.4:** Tightened normative language (promoted key SHOULD→SHALL), added Deviation Evidence requirements, clarified Environment vs Injection boundaries, added explicit “Allowed Places for Side Effects”, added cancellation/task-handling standard, added modal/alerts routing rules, added SwiftData migration guidance, added Networking conventions, expanded motion accessibility requirements and fallbacks, added Forbidden Patterns annex and PR Evidence annex.

---

## Foreword
This standard defines a consistent approach for building maintainable, testable, accessible, and performant SwiftUI applications on iOS 17+. It uses ISO-like language (“SHALL/SHOULD/MAY”) to establish clear engineering expectations.

## Introduction
SwiftUI enables fast iteration but can become inconsistent without strict rules around state ownership, concurrency, navigation, motion, and side effects. iOS 17 adds powerful tools (Observation, improved transitions, symbol effects, keyframe/phase animation, and sensory feedback). This standard ensures these capabilities are applied predictably and responsibly.

---

## 1. Scope
This standard specifies requirements for:
- Project organization and naming
- SwiftUI composition and state ownership (Observation-first)
- Concurrency, cancellation, and error state modeling
- Navigation patterns (including sheets/alerts)
- Persistence (SwiftData baseline) and networking boundaries
- Interaction & motion: gestures, animations, transitions, matched geometry, haptics
- Accessibility, localization, privacy, and release readiness
- Testing, previews, and performance discipline
- Review compliance and deviation handling

---

## 2. Normative references
- Apple Developer Documentation: SwiftUI, Observation, Swift Concurrency, NavigationStack
- Apple Human Interface Guidelines (HIG)
- App Store Review Guidelines
- Accessibility guidance aligned with WCAG principles as applicable to mobile UI

---

## 3. Terms and definitions
**SHALL** = mandatory requirement
**SHOULD** = recommended (deviation requires justification)
**MAY** = optional

**Feature**: cohesive user-facing capability (e.g., Search, Profile)
**Feature Model**: UI-facing state + user intents for a feature
**Service**: external boundary (network, persistence, analytics)
**Route**: typed navigation destination representation
**Deviation Evidence**: required PR documentation when not meeting SHALL/SHOULD (see 18.2)

---

## 4. General principles

### 4.1 Maintainability
- Code SHALL optimize for readability and changeability over micro-optimizations.
- Each feature SHALL be independently previewable and testable.

### 4.2 Single source of truth
- Each piece of mutable state SHALL have one clear owner.
- State SHALL flow downward; user intents SHALL flow upward (data-down, actions-up).
- Views SHALL NOT maintain “shadow state” that duplicates model state (except transient presentation state such as focus/scroll position).

### 4.3 Determinism and explicit UI states
- Async screens SHALL model UI states explicitly (idle/loading/loaded/empty/failed).
- Multiple booleans for “loading/error/empty” SHALL NOT be used when state combinations can become inconsistent.
- Side effects SHALL be orchestrated by Feature Models and Services, not by Views.

### 4.4 Platform availability discipline (iOS 17.0 minimum)
- All APIs used SHALL be available on iOS 17.0 unless guarded by `if #available(...)`.
- Any guarded path SHALL include a tested fallback behavior and a preview/demo path.

---

## 5. Project structure and naming

### 5.1 Directory structure (baseline)
- `App/`
  - `AppEntry.swift`
  - `AppEnvironment.swift` (dependency container / composition root)
  - `Routing/` (shared routes if needed)
- `Core/`
  - `Models/`
  - `Services/` (protocols + implementations)
  - `DesignSystem/` (tokens + reusable components)
  - `Utilities/` (formatters, helpers)
- `Features/`
  - `FeatureName/`
    - `FeatureNameView.swift`
    - `FeatureNameModel.swift` (Observation model)
    - `FeatureNameRoutes.swift` (optional)
    - `Components/`

### 5.2 Naming
- Views SHOULD end with `View`.
- Feature models SHALL end with `Model`.
- Service protocols SHALL end with `Servicing` (preferred) or `Service` (allowed); the choice SHALL be consistent across the app.

---

## 6. Architecture and responsibilities

### 6.1 View responsibilities
SwiftUI Views:
- SHALL render based on provided state.
- SHALL forward user actions to a Feature Model (intent methods) or injected handlers.
- SHALL NOT perform networking, persistence, analytics, logging, or business rules directly.
- SHALL keep `body` concise by extracting subviews/builders when layout becomes non-trivial.
- SHALL NOT create long-lived background work except via `.task` at the feature root (see 6.4).

### 6.2 Feature Model responsibilities
Feature Models:
- SHALL own UI state for a feature.
- SHALL expose intent methods (e.g., `refresh()`, `didTapSave()`).
- SHALL coordinate with services for side effects.
- SHALL be `@MainActor` when mutating UI-driving state (default expectation).
- SHALL be testable without SwiftUI (no View references inside models).

### 6.3 Services boundary
- All external interactions SHALL be abstracted behind service protocols.
- Services SHALL be injected (initializer injection preferred).
- Services SHALL be deterministic and mockable in tests.

### 6.4 Allowed places for side effects (strict)
- Views MAY start work ONLY by calling Feature Model intents.
- Only the feature root view (e.g., `FeatureNameView`) MAY initiate `.task` or `.onAppear` that triggers a model intent.
- Subviews SHALL NOT start side effects (no `.task`, `.onAppear`, `.onChange` that triggers services) unless explicitly documented as a standalone feature root.
- Views SHALL NOT call services directly (even if accessible via environment).

---

## 7. State management (iOS 17+ Observation)

### 7.1 Required approach
For iOS 17+ projects:
- Feature Models SHALL use Observation (`@Observable`) rather than `ObservableObject` unless justified.
- Views SHALL own feature models with `@State` when created locally.
- Child views SHOULD use `@Bindable` only when they need to mutate safe, local UI properties.

### 7.2 Source-of-truth rules
- Shared app dependencies MAY be accessed via `@Environment` using an AppEnvironment/container type ONLY at composition roots.
- `@EnvironmentObject` SHOULD be avoided unless the object is truly global, stable, and widely shared.
- Feature Models SHALL receive dependencies via initializer injection; environment lookup inside Feature Models is NOT allowed.

### 7.3 UI state modeling
- Async screens SHALL use an explicit UI state representation.
- UI state SHALL be represented as:
  - an enum with associated values, or
  - a single struct with a mutually-consistent state machine (documented).
- UI SHALL never render “impossible states” (e.g., loaded + error simultaneously).

### 7.4 Mutation policy (prevent “Bindable chaos”)
- Properties that can trigger side effects (network/persistence/analytics) SHALL NOT be mutated directly via `@Bindable`.
- Side-effecting changes SHALL go through intent methods.
- `@Bindable` direct mutation is allowed for:
  - form fields,
  - local toggles that do not trigger external work,
  - transient UI-only state (presentation).

---

## 8. Concurrency and async work

### 8.1 MainActor discipline
- UI-driving state SHALL be mutated on the main actor (typically via `@MainActor` feature models).
- Manual `DispatchQueue.main.async` SHALL NOT be used as the primary concurrency mechanism.

### 8.2 Cancellation (mandatory)
- Long-running operations SHALL be cancellable.
- View-triggered loads SHOULD use `.task(id:)` where identity-based cancellation is desired.
- Feature models SHALL prevent duplicate in-flight requests for the same resource unless explicitly required.

### 8.3 Error handling
- Errors SHALL map to user-facing states and messages.
- Raw error strings SHALL NOT be shown directly to end users.
- Errors SHOULD be modeled as typed domain errors, not generic `Error`.

### 8.4 Task handling pattern (standardized)
- Feature Models that perform async work SHALL:
  - store in-flight `Task` handles (or use structured concurrency patterns) and cancel on new conflicting intent,
  - cancel in-flight work when the feature is no longer active (e.g., `onDisappear` intent or explicit `deinit` cancellation strategy),
  - ensure each intent produces a predictable state transition path (testable).
- Repeating work (timers, async sequences, polling) SHALL stop on cancellation and shall respect app lifecycle where applicable.

---

## 9. Navigation

### 9.1 NavigationStack baseline
- Apps SHALL use `NavigationStack`.
- Non-trivial flows SHALL use typed routes (e.g., `enum Route: Hashable`).

### 9.2 Predictable navigation side effects
- Navigation SHALL be driven by state/routes (e.g., path, selectedRoute), not hidden side effects inside view closures.
- Deep link parsing (if applicable) SHALL be unit tested independently of UI.

### 9.3 Sheets, alerts, and transient navigation
- Sheets, full-screen covers, and alerts SHALL be driven by explicit state (e.g., `enum Modal`, `AlertState`).
- Ad-hoc `@State var showSheet = false` in deeply nested subviews SHALL be avoided; the feature root (or feature model) SHALL own presentation state for the feature.
- If multiple presentations can occur, they SHALL be mutually exclusive by construction (single owner state machine).

---

## 10. Persistence (SwiftData recommended baseline)

### 10.1 SwiftData usage
- SwiftData MAY be used as the default persistence layer for iOS 17+.
- Persistence access SHALL be encapsulated behind a repository/service boundary for testability.
- Alternative storage choices (Core Data, SQLite, files) SHALL be documented when used.

### 10.2 Query discipline
- `@Query` SHOULD be used for simple fetch-driven UI.
- Complex derivations SHALL be computed outside `body` when they affect readability/performance.
- Queries SHALL NOT trigger heavy computation repeatedly during view updates.

### 10.3 Schema evolution / migrations (mandatory when persistence exists)
- Any schema change SHALL include:
  - a migration plan (even if “wipe & rebuild” for non-user data),
  - a test strategy (unit/integration),
  - rollback/forward considerations.
- Persisted user data SHALL NOT be silently destroyed without product approval.

---

## 11. Networking

### 11.1 API client boundary
- Networking SHALL be performed through an API client/service, not in Views.
- Transport models SHALL be decoupled from UI state.
- Network calls SHALL be representable in tests via mocks/stubs and fixture responses.

### 11.2 Resilience and UX
- Retry behavior SHALL be explicit and user-driven for most UI flows.
- Offline/error scenarios SHALL render stable UI (no blank screens).
- Loading states SHALL be explicit (no “hidden spinner by animation”).

### 11.3 Auth, caching, and logging (baseline expectations)
- Auth token management SHALL be centralized (not scattered across features).
- Caching policy SHALL be defined per endpoint/resource where applicable (none/default/TTL).
- Logging SHALL redact sensitive data; request/response logging SHALL be gated by build configuration.

---

## 12. Interaction & Motion (Gestures, Animation, Transitions, Matched Geometry) — iOS 17+

### 12.1 Principles
- Motion SHALL be purposeful (communicate state change, hierarchy, or feedback).
- Motion SHALL be accessible (respect Reduce Motion and avoid disorienting effects).
- Interaction SHALL be predictable (gestures must not break standard system behaviors).

### 12.2 Gesture standards
- Prefer standard controls (`Button`, `Toggle`, `NavigationLink`) over custom gestures for accessibility and semantics.
- Gestures SHALL be attached at the smallest scope that achieves the behavior.
- Gestures SHALL NOT interfere with primary scrolling unless explicitly intended and documented.
- Conflicts SHALL be resolved intentionally using `simultaneousGesture` / `highPriorityGesture` only when necessary.
- Critical actions SHALL NOT rely solely on non-obvious gestures without visible affordances.

### 12.3 Animation standards (general)
- Animations SHOULD be driven by state changes rather than imperative timelines.
- Use explicit animation (`withAnimation`) for user-intent transitions; `.animation(_, value:)` SHALL be narrowly scoped and SHALL NOT be applied at feature-root container level without justification.
- Animations SHALL NOT be used to mask loading delays; loading states must be explicit.
- Animations SHOULD be interruptible (avoid “locked” sequences that ignore user input).

### 12.4 iOS 17 animation APIs (recommended)
When appropriate, teams SHOULD use iOS 17 animation tools where they reduce complexity and improve semantics:
- `phaseAnimator` for phase/state-driven animations
- `keyframeAnimator` for multi-step/timeline-like animations
- SF Symbols `symbolEffect` for icon feedback
- `sensoryFeedback` for haptic confirmation tied to state triggers

Rule:
- These APIs SHALL NOT be introduced “just because”; they must simplify code and improve clarity versus simpler animations.

### 12.5 Motion accessibility requirements (mandatory)
- UI SHALL respect Reduce Motion:
  - read `@Environment(\.accessibilityReduceMotion)` (or equivalent) and reduce/disable heavy motion.
- Any “travel” animations (large movement across the screen), parallax-like effects, and repeating/continuous animations SHALL be disabled or replaced when Reduce Motion is enabled.
- High-motion patterns SHALL be replaced by:
  - subtle fades,
  - opacity/scale changes,
  - direct state changes without animated travel.
- Motion SHALL NOT be the only indicator of a state change (also use text, icon, color, or layout cues).

### 12.6 Transitions and content changes
- Transitions SHALL be used consistently to communicate hierarchy changes (insert/remove, expand/collapse, screen state changes).
- Prefer simple transitions unless UI meaning requires complexity.
- iOS 17 enhancements MAY be used where they improve comprehension:
  - content transitions for value changes (e.g., numeric/text changes),
  - scroll transitions sparingly and only when reinforcing structure.

### 12.7 Matched geometry (`matchedGeometryEffect`) standards
- Matched geometry MAY be used to preserve spatial continuity between two representations of the same element.
- Namespaces SHALL be owned by the parent coordinating the transition.
- Matched elements SHALL have stable identity.
- Matched geometry SHALL NOT be used in lists/grids where identity is unstable/reused or where it causes layout glitches.

Requirements:
- Matched geometry transitions SHALL be validated in:
  - large Dynamic Type sizes,
  - dark mode,
  - Slow Animations (developer setting) to verify correctness.
- When matched geometry causes fragility/glitches, the team SHALL prefer simpler transitions over fragile hero animations.

### 12.8 Haptics and feedback (iOS 17 `sensoryFeedback`)
- Haptics SHOULD be tied to meaningful user actions (commit, success, error).
- Haptics SHALL NOT be tied to continuous updates (scroll position, drag progress, rapid toggles) unless throttled and explicitly justified.
- Feedback SHALL correspond to state triggers (not timers).

### 12.9 Performance rules for motion
- Avoid animating expensive layout changes across large hierarchies.
- Avoid starting many simultaneous animations in lists/grids.
- Prefer animating opacity/transform over frequent layout invalidation when possible.
- Heavy effects (blur chains, complex masks) SHOULD be limited and validated on older supported devices.

---

## 13. SwiftUI composition and performance

### 13.1 View size and extraction
- Views SHOULD be decomposed when `body` becomes large (guideline: > ~60 lines of layout).
- Prefer subviews and `@ViewBuilder` helpers over deep nesting.

### 13.2 Identity and lists
- `List`/`ForEach` SHALL use stable identifiers.
- Index-based identity SHALL be avoided unless strictly required and documented.

### 13.3 Avoid type erasure by default
- `AnyView` SHALL NOT be a routine tool; prefer builders and composition.

---

## 14. Accessibility and localization

### 14.1 Accessibility
- Controls SHALL have correct accessibility labels and traits where needed.
- Dynamic Type SHALL be supported.
- Color SHALL NOT be the only indicator of meaning.
- Custom gesture-only interactions SHALL provide accessible alternatives and discoverable affordances.

### 14.2 Localization
- User-facing strings SHALL be localized.
- Dates/numbers/currency SHALL use locale-aware formatting.
- String interpolation SHALL be localized safely (no concatenated sentences that break grammar).

---

## 15. Design system usage
- Spacing, typography, colors, radii SHOULD come from a centralized design system.
- Magic numbers SHOULD be minimized.
- New reusable components SHALL be added to the design system rather than copied across features.

---

## 16. Logging, analytics, privacy
- Sensitive data SHALL NOT be logged.
- Analytics SHALL be routed via an analytics service; events SHOULD be defined centrally.
- Data collection SHALL follow least-privilege and disclosed privacy requirements.

---

## 17. Testing requirements

### 17.1 Unit tests (mandatory for logic)
- Feature model logic and state transitions SHALL be unit tested.
- Any feature with async work SHALL include tests for loading → loaded/empty/failed paths.
- Route parsing/mapping SHALL be unit tested where deep links or typed routing exists.

### 17.2 UI tests
- Critical user flows SHOULD have UI test coverage (minimum smoke tests).

### 17.3 Previews (mandatory)
Each feature view SHALL include previews for:
- loaded
- empty
- error
Additionally (recommended but expected for motion/complex UI):
- loading
- dark mode
- large Dynamic Type
- Reduce Motion behavior (where motion exists)

---

## 18. Code review and compliance

### 18.1 Review checklist (minimum, mandatory)
Reviewers SHALL confirm:
- single source of truth for mutable state (no shadow state)
- explicit async UI state (idle/loading/loaded/empty/failed)
- no networking/persistence/analytics in Views
- side effects initiated only at feature root via model intents
- MainActor-safe UI state updates
- cancellable async work (no leaks, no duplicate requests unless documented)
- stable list identity
- accessibility basics (labels, dynamic type, semantics)
- localization readiness
- motion rules followed (Reduce Motion fallback, no gesture conflicts, no fragile matched-geometry misuse)
- previews and tests added where required
- iOS 17 availability compliance (no accidental iOS 18-only APIs without guards)

### 18.2 Deviations (strict evidence required)
- Deviations from **SHALL** require Deviation Evidence in PR description.
- Deviations from **SHOULD** require a brief rationale in PR description if they impact maintainability/performance/accessibility.

Deviation Evidence MUST include:
1) What requirement is being deviated from (section number).
2) Why the deviation is necessary.
3) Alternatives considered and why rejected.
4) Risk assessment (user impact, maintainability).
5) Mitigation plan (tests, monitoring, follow-up ticket if needed).

---

## Annex A — Definition of done (suggested)
A feature is done when it:
- meets acceptance criteria
- includes required previews for major states
- includes unit tests for logic/state transitions (especially async paths)
- handles loading/empty/error deterministically
- passes accessibility smoke checks
- respects Reduce Motion if it uses significant motion
- passes review checklist

---

## Annex B — Motion “Do/Don’t” quick rules (non-normative)

**Do**
- animate meaningful state changes
- use symbolEffect/sensoryFeedback for crisp micro-interactions
- keep transitions simple and consistent
- provide Reduce Motion fallbacks

**Don’t**
- animate everything
- block scroll with parent gestures accidentally
- rely on matched geometry when identity/layout are unstable
- use motion as the only state indicator
- produce “haptic spam” on rapid updates

---

## Annex C — Forbidden patterns (normative)
The following patterns SHALL NOT be used:
- Networking, persistence, analytics, or business rules executed directly in Views.
- Services pulled ad hoc from environment in subviews (“service locator by stealth”).
- Multiple booleans representing one conceptual screen state (loading/error/empty drift).
- `.animation(_, value:)` applied at feature-root container scope without Deviation Evidence.
- Matched geometry in unstable-identity lists/grids or where it causes layout glitches.
- Continuous/repeating animations without Reduce Motion fallback and cancellation.
- Gesture layers that block scrolling/standard behaviors without explicit documented intent and validation.

---

## Annex D — PR Evidence requirements (normative)
PRs that include any of the following MUST attach evidence:
- Motion/gesture changes:
  - short screen recording (normal) AND Reduce Motion mode recording
  - note any gesture conflict testing (scroll + VoiceOver basic check)
- Navigation flow changes:
  - screenshot(s) of route states (or brief description of route ownership)
- Persistence schema changes:
  - migration plan summary + tests run
- Significant async work:
  - test list (which state transitions are covered)

End of document.
