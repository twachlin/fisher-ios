# iOS App Architecture Guide

This document captures the current architectural decisions for this iOS
application. It is intended to guide future AI-assisted changes so the project
grows with clear responsibilities, stable naming, and explicit decision points.

## Purpose

- Preserve a clean separation between `data`, `domain`, and `presentation`.
- Keep the app scalable while it grows feature by feature.
- Prefer explicit structure over convenience-driven coupling.
- Reevaluate architecture when growth justifies it, but always explain why and
  ask for approval before making structural changes.
- Use modern Apple-native technologies as the baseline for implementation:
  `SwiftUI`, `Observation`, `Swift Concurrency`, `SwiftData`, and
  `Swift Testing`.

## Platform Baseline

- Prefer `iOS 17+` for new development unless explicit product constraints
  require support for older OS versions.
- Use `SwiftUI` as the default UI framework.
- Use `Observation` with `@Observable` for presentation state unless a specific
  interoperability reason requires `ObservableObject`.
- Use `async/await` and structured concurrency as the default async model.
- Use `SwiftData` as the default local persistence solution when persistence is
  needed.
- Use `Swift Testing` for new tests.
- Use `Swift Package Manager` for dependencies.
- Use `UIKit` only when a platform capability is unavailable or unjustifiably
  difficult in pure SwiftUI.

## Product Reference Rule

This iOS project is the platform counterpart of the Android app.

The Android project is the current functional source of truth for product
behavior, user flows, feature scope, backend contract usage, and visible app
semantics unless product requirements explicitly state otherwise.

This does not mean iOS must mirror Android implementation details literally.

When implementing features on iOS:

1. review the equivalent flow in the Android project
2. identify the real functional behavior and backend contracts
3. reproduce the same product outcome on iOS
4. use iOS-native architecture, APIs, and platform conventions to implement it

If Android behavior appears inconsistent, ambiguous, or tightly coupled to an
Android-specific implementation detail, stop and call it out before copying it
into iOS.

## Core Principles

- Treat `data`, `domain`, and `presentation` as if they were independent
  modules even if the project starts as a single target.
- Do not expose `data` models or infrastructure types outside the `data` layer.
- Keep business rules in `domain` or in explicitly justified presentation
  orchestration, not inside views.
- Keep SwiftUI views focused on rendering state and forwarding user actions.
- Prefer small, purpose-specific files and types.
- Avoid unnecessary line breaks in code throughout the project. Prefer compact,
  readable formatting unless a line break materially improves clarity.
- Import project types in the import section and reference them by simple name
  in code bodies. Do not use fully qualified names inline unless there is a
  real naming collision that cannot be resolved cleanly with imports.
- Never add extracted SDK documentation, generated symbol dumps, decompiled
  content, or other inspection-only artifacts into the project tree.

## App Composition Strategy

The app should use a SwiftUI-first lifecycle.

This is intentional.

The app entry point must stay thin and act only as the host for the root app
composition. It should not accumulate navigation logic, authentication
decisions, or feature orchestration.

### Responsibility Split

- `App`
  Declares the app lifecycle and root scene only.
- `AppEntry`
  Root view that decides which high-level app experience to render.
- `AppLaunchViewModel`
  Owns startup and session-entry state.
- `AuthenticationEntry`
  Entry point for the authentication area.
- `MainNavigation`
  Root authenticated navigation container.

### MVP App Areas

The initial MVP starts with authentication. After the user is authenticated,
the app exposes three root sections:

- `Huntcast`
- `Map`
- `More`

These sections are the current authenticated navigation pillars. Future areas
may be added only when product scope justifies them.

## Naming Conventions

Use names that describe architectural responsibility, not generic framework
patterns.

### Preferred names

- `Entry`
  Use for a root decision point or entrance into a bounded app area.
- `Navigation`
  Use for a container responsible for route composition and navigation
  structure.
- `Screen`
  Use for a container view connected to a state owner.
- `Content`
  Use for presentational views that render state and emit callbacks.
- `ViewModel`
  Use for the owner of UI state and presentation orchestration.
- `Store`
  Use only when the type truly represents a state container or persistence
  store, not as a synonym for any view model.

### Examples

- `AppEntry`
- `AppLaunchViewModel`
- `AuthenticationEntry`
- `MainNavigation`
- `LoginScreen`
- `LoginContent`
- `HuntcastScreen`
- `MapScreen`
- `MoreScreen`

### Avoid when a more semantic name exists

- `AppRoot`
- `AuthRoot`
- `MainShell`
- names that look like global singleton services
- names that can be confused with `AsyncSequence`, `Task`, or reactive streams

## Presentation Layer Rules

- Every screen should define explicit presentation contracts:
  - `UiState`
  - `UiAction` or `UiEvent`
  - `UiEffect` when one-off events are needed
- `UiState` must be immutable.
- The screen state owner is the single source of truth for screen state.
- Prefer `@Observable` state owners for modern SwiftUI flows.
- Domain models must be converted into presentation-specific `UiModel`s before
  reaching reusable visual components.
- Reusable visual components must receive only:
  - `UiModel`s
  - primitives or simple values
  - callbacks
- If a presentation need cannot be implemented cleanly because a required
  `data` or `domain` contract is missing, stop and propose the missing contract
  before implementing a UI workaround.
- UI-visible text must not be hardcoded in views or presentation logic.
- All UI text must come from `Localizable.strings` or an equivalent localized
  resource contract.
- Keep view modifiers, bindings, and navigation wiring readable. If a view body
  becomes too large, split it into focused presentational components.

### SwiftUI State Rules

- Prefer `@State` and `@Environment` with Observation-based models where
  appropriate.
- Avoid defaulting to `ObservableObject`, `@StateObject`, and
  `@EnvironmentObject` in new code unless older deployment targets or specific
  interoperability constraints require them.
- Keep long-running or cancelable work coordinated from the presentation state
  owner, not embedded ad hoc in views.

### Sheet And Full-Screen Presentation Rules

- When a screen hosts multiple sheets or full-screen covers, keep a single host
  presentation decision in the screen container.
- Prefer a typed presentation state instead of multiple unrelated booleans.
- Action objects may be scoped by presentation responsibility instead of using
  one long flat list of callbacks.
- Do not introduce one large shared actions object that mixes unrelated sheet
  behaviors.
- Keep action object definitions separate from their construction.
- Prefer dedicated helpers when callback wiring becomes large, but keep helpers
  focused on one feature family.

## Domain Layer Rules

- `domain` must not depend on `SwiftUI`, `UIKit`, or Apple UI framework types.
- `domain` must not expose `data` models, DTOs, repositories, or transport
  result types.
- Domain models must be distinct from `data` models even when fields are
  similar.
- Keep feature contracts in feature-scoped groups such as:
  - `domain/feature/<feature>/model`
  - `domain/feature/<feature>/repository`
  - `domain/feature/<feature>/usecase`
- Keep shared contracts in common groups such as:
  - `domain/common/result`
  - `domain/common/error`

### Domain Naming Rules

- Domain models must use names that reflect product meaning rather than API
  transport shape.
- Use cases should be named as clear business operations.
- Repository protocols in `domain` should describe required capability, not
  implementation source.
- Avoid suffixes such as `Impl`, `Manager`, or `Helper` unless the role is
  genuinely infrastructure-oriented and no better domain name exists.

### Domain Logic Placement Rules

- Put business rules in `domain` when they represent product semantics and can
  be expressed independently of UI and transport concerns.
- If a rule depends on feature semantics rather than transport concerns, prefer
  implementing it in `domain` or at the `data -> domain` boundary.
- If logic currently lives in `data` but is actually domain behavior, call it
  out and move it only when the behavior is well understood.
- Keep use cases focused by single responsibility.

## Data Layer Rules

- `data` owns transport, persistence, session storage, and infrastructure
  concerns.
- `data` may depend on Apple frameworks or infrastructure libraries when
  justified.
- `data` must not leak DTOs, persistence models, transport errors, or storage
  implementation details outside its boundary.
- Keep `data` organized by responsibility, with feature-scoped APIs and
  mappers.

### Data Package Structure

Prefer this structure:

- `data/remote`
- `data/local`
- `data/repository`
- `data/model`
- `data/mapper`
- `data/network`
- `data/di` or `data/dependencies`

Use feature-specific subgroups when the feature is large enough to justify
them.

### Remote Rules

- Keep shared HTTP client creation in a single remote foundation area.
- Define APIs per feature instead of one large service interface.
- Prefer names such as:
  - `AuthApi`
  - `HuntcastApi`
  - `MapApi`
  - `MoreApi`
- Remote response models must end with `Response`.
- Request payload models should use descriptive names such as `...Request`.
- Keep response-envelope parsing close to the repository or in clearly shared
  infrastructure helpers.

### Data Model Rules

- Data models must be distinct from domain models.
- Data models should represent app-facing data inside the `data` boundary, not
  raw transport.
- Do not reuse domain model names in `data`.
- Feature-specific mapping should happen in feature mappers, not in networking
  clients.

### Repository Rules In Data

- Data repositories are responsible for coordinating remote and local sources.
- Repositories may parse envelopes, map DTOs, persist session or local data,
  and adapt infrastructure errors.
- Keep repository files focused on one feature or one bounded responsibility.
- Do not place business rules in `data` when they belong in `domain`.
- If a rule is temporarily implemented in `data`, call it out explicitly.

### Network And Error Rules

- Prefer `URLSession` as the default networking stack unless there is a strong
  reason to adopt a third-party abstraction.
- Keep transport-layer failures inside `data` using shared transport result
  types.
- Prefer shared transport contracts such as `NetworkResult` and `AppError`
  inside `data`.
- `AppError` should remain transport and infrastructure-focused and must not
  become the public error model of `domain`.
- Authenticated headers must be resolved through shared infrastructure, not
  appended ad hoc across features.

### Session Rules

- Session persistence belongs to `data`.
- The minimum persisted authenticated session data is:
  - `token`
  - `user_id`
- Keep session storage behind interfaces such as:
  - `SessionStore`
  - `SessionSnapshotProvider`
- Store secrets in `Keychain` when appropriate.
- Do not store sensitive session data in plain `UserDefaults`.

### Local Persistence Rules

- `SwiftData` is the official local persistence solution for this project.
- Introduce local persistence incrementally and only when a feature needs it.
- Do not mix remote DTOs with persistence models.
- Do not expose persistence models outside `data`.
- If a requirement cannot be satisfied cleanly with `SwiftData`, explain the
  constraint before introducing another persistence approach.

## Dependency Management And Tooling

- Use `Swift Package Manager` for dependencies.
- Prefer constructor injection and explicit dependency passing.
- Keep dependency wiring focused on infrastructure setup, not business policy.
- Avoid hidden singleton coupling for feature dependencies.
- If a dependency container is introduced, keep it simple, typed, and explicit.

## Navigation Rules

- Keep top-level navigation orchestration in `presentation/navigation`.
- Use `NavigationStack` as the default foundation for navigation.
- Keep feature-internal navigation close to the feature when it grows enough to
  justify it.
- Do not couple small reusable components to navigation APIs.
- Navigation decisions belong in entry points, navigation containers, or screen
  containers.
- `MainNavigation` owns only authenticated app-level navigation concerns:
  selected section, scaffold or chrome, and truly global destinations.
- Each root app section must own its internal navigation in a feature-scoped
  `.../navigation/<Section>Navigation.swift` container instead of defining all
  internal routes inside `MainNavigation`.
- Section-specific routes should live with their feature navigation group, not
  in shared app navigation files.
- Destinations reachable from multiple sections may remain global, but the
  decision to open them should be delegated from the section navigation rather
  than re-centralizing all section routing in `MainNavigation`.

### MVP Navigation Structure

- Authentication flow belongs to the authentication feature area.
- Authenticated root navigation starts after a valid session exists.
- The authenticated root starts with the three MVP sections:
  - `Huntcast`
  - `Map`
  - `More`
- If one of these sections grows into a multi-route area, create a dedicated
  feature navigation container for it instead of expanding the root container
  indefinitely.

## Testing Rules

- Use `Swift Testing` for new unit and integration tests by default.
- Keep domain use cases highly testable and covered first.
- Presentation logic should be testable without rendering UI whenever possible.
- Use snapshot or UI tests only where they provide clear value, not as a
  substitute for domain and presentation logic tests.
- Keep test fixtures feature-scoped and readable.

## Feature Growth Guidance

When a feature grows, prefer this progression:

1. `Screen` + `Content`
2. explicit `UiModel`
3. dedicated mapper or adapter in `presentation`
4. feature-scoped navigation container when the feature has multiple routes

Do not introduce extra layers unless there is a clear scaling reason.

## Architectural Reevaluation Policy

As the project grows, architecture can be revisited.

That reevaluation must be explicit.

When proposing a structural change, always:

1. explain what is no longer scaling well
2. describe the tradeoff in the current structure
3. propose the new structure
4. explain why it is better now than before
5. ask for approval before applying the change

Examples of valid reevaluation triggers:

- `MainNavigation` becoming too large
- a feature needing its own internal navigation graph
- startup or session logic expanding beyond what `AppLaunchViewModel` should
  own
- repeated naming or package conventions no longer fitting the codebase

## Feature Implementation Workflow

Before implementing a new feature or expanding an existing one:

1. review the equivalent flow in the Android project
2. identify the real functional behavior and backend contracts
3. explain the proposed iOS structure and tradeoffs
4. implement the feature by layer, keeping boundaries explicit

### Incremental Delivery Rule

- Prefer building features from lower to higher architectural layers:
  1. `data`
  2. `domain`
  3. `presentation`
- Do not jump to `presentation` before the feature contracts in lower layers
  are stable enough.

### Complexity Progression Rule

- Prefer implementing features from lower to higher difficulty.
- Start with clearer, bounded flows before highly stateful or highly coupled
  areas.

## Decision Rule For Future AI Changes

If a change would alter:

- root app composition
- startup or session entry behavior
- navigation boundaries
- ownership of business rules between `domain` and `presentation`
- naming conventions for root architectural elements
- the MVP authenticated root sections

do not implement it silently.

Explain the proposal and ask for agreement first.
