# ARCHITECTURE GOSPEL — CLEAN ARCHITECTURE

The shared Clean Architecture canon for Shippopotamus Android apps.
This document holds the general rules every app follows. Each repo's
own architecture contract holds that repo's module map, dependency
graph, package namespaces, and exceptions. Where a repo contract and
this gospel disagree on a general rule, this gospel wins. Where a repo
contract records a local exception, the exception stands and says so.

Synthesized from the architecture contracts of Stratum, Bloviate, and
Enclave, which state these rules in near-identical words.

---

## 0. Clean Architecture Supremacy Rule (ZERO TOLERANCE — HIGHEST PRIORITY)

Every Shippopotamus app follows Clean Architecture / ports-and-adapters.
Dependencies point inward toward the domain layer; data, service, and
presentation code adapts framework, storage, platform, and UI concerns
around domain ports.

**The agent MUST NEVER:**

- Write, propose, or allow any code that violates the dependency rule.
- Leak Android, Compose, Hilt, Room, WorkManager, or any other
  framework/SDK implementation types through domain public APIs.
- Bypass layer boundaries, forbidden imports, or layer responsibilities.
- Use any other architectural pattern (MVVM-only, monolithic, etc.).
- Put business logic in the app shell or in composables.
- Pass domain models directly to composables.
- Pass `NavController` into composables.

**Violation = immediate contract breach.**
Reply with exactly: "Contract violation detected in [file]. Request explicit 'Contract Waiver' from user."
Then stop. Do not generate code until waived.

All rules below are non-negotiable implementations of Clean Architecture.

---

## Strict Boundaries

| Always | Never |
|--------|-------|
| Route background progress via `WorkManager.setProgress`. | Use singleton callbacks or static listeners for background-to-UI updates. |
| Use `ExistingWorkPolicy.KEEP` or `REPLACE` for unique work. | Scatter multiple unmanaged `OneTimeWorkRequests` for the same task. |
| Implement byte-offset resumption for IO tasks. | Assume a background task will run to completion in a single session. |
| Route navigation via pure callbacks hoisted to the route level. | Pass `NavController` into Composables. |
| Expose a single `StateFlow<UiState>` from ViewModels. | Expose mutable flows or Android framework types from ViewModels. |
| Ensure each screen owns its own `Scaffold` and `TopAppBar`. | Thread `paddingValues` down the UI tree from a root-level Scaffold. |
| Consume one-shot events via `Channel<Event>` and `LaunchedEffect`. | Use `SharedFlow` for one-shot UI events (can cause dropped events). |
| Use stable keys on all `LazyColumn` and `LazyRow` items. | Import data, service, presentation, app, feature, UI, or native-bridge code into the domain layer. |
| Enforce Clean Architecture dependency inversion at module boundaries. | Allow reverse dependencies into the domain layer. |
| Keep domain APIs as domain models, ports, use cases, and test fixtures. | Expose Android/Compose/Hilt/Room or other framework types from domain APIs. |

---

## 1. Dependency Direction (Strict)

- Dependencies point inward. Domain depends on nothing else in the app:
  no app, feature, data, UI, service, presentation, or native-bridge
  modules, and no Android framework types in its public APIs.
- Data, UI, service, presentation, and feature code may depend on
  domain. Application/use-case layers sit between domain and the outer
  layers and depend on domain only.
- Data modules implement domain (or application) ports and never import
  the app shell or presentation code.
- Feature modules depend on domain and shared UI, and on data only
  where the repo's contract declares it. Feature modules never depend
  on other feature modules; cross-feature navigation is wired in the
  app shell via callbacks and shared routes.
- The DI package/module is the composition root: the only place ports
  meet implementations.
- External libraries are consumed behind a port owned by an inner
  layer; library types do not leak into domain, application, or
  presentation APIs.

**No reverse dependencies into the domain layer are allowed.** Each
repo's architecture contract carries its exact module graph, and the
graph is enforced by Gradle wherever the module split exists.

**Ports belong to the layer that consumes them.** A port consumed by
use cases or services lives with that layer; a port consumed only by a
domain service/evaluator stays in the domain layer. When a layer is
extracted into its own module, its ports move with it.

---

## 2. Forbidden Imports (Principles)

| Layer | Must NOT import |
|-------|-----------------|
| Domain | Application, data, service, presentation, app, feature, UI, or native-bridge code; Compose; Hilt annotations on domain types; Android framework types in public APIs; library implementation classes |
| Application / use cases | Data, service, presentation; Compose; Android framework types; library implementation classes |
| Data | Compose UI; service; presentation; the DI package; anything in the app shell |
| Presentation | Data classes and library implementation types — go through use cases and ports |
| Feature | Other feature modules |

Reusable UI code lives with the presentation layer or the shared UI
module; screens are wired by the app shell.

---

## 3. Layer Responsibilities

- **Domain** — Domain models, evaluators/policies, domain-only ports,
  use cases/interactors, diagnostics, and test fixtures. Pure
  Kotlin/JVM, framework-free. Keep domain APIs free of
  Android/UI/storage/native implementation types.
- **Application** (where the repo splits it out) — Use cases,
  application ports (ports consumed by use cases or services), and
  port-chaining processors. Depends only on domain.
- **Data** — Implements persistence, downloads, repositories, network
  integration, and platform adapters behind ports. Entity ↔ domain
  mapping lives here. No Compose UI.
- **Service** (where present) — Long-lived platform services. Services
  are thin: they translate platform events into use-case calls and
  render state.
- **Presentation / shared UI / features** — Compose UI and
  presentation logic. ViewModels map domain → presentation models.
- **App shell / DI** — Application class, navigation wiring, DI
  composition, app resources, platform entrypoints. No business rules —
  only wiring and composition.

### 3.1 Domain model policy: rich aggregates, honest projections

Types in the domain layer fall into four classes, and the class decides
how much behavior a type must carry:

- **(A) Aggregates / domain models — rich.** We own the concept and are
  authoritative about its validity. They enforce their invariants at
  construction (`init` or factory — `copy()` runs the constructor, so
  `init` checks cannot be bypassed) and expose state transitions as
  named methods on the type, never as raw `copy()` state edits in a
  use case.
- **(B) Value objects — rich and small.** Immutable, validated at
  construction, behavior attached. A collection that carries rules is
  wrapped as a value object, not passed as a bare list.
- **(C) Domain services — procedural by design.** Rules that span
  aggregates or need full history live here (engines, calculators,
  response modeling). They do not move onto an arbitrarily chosen
  aggregate.
- **(D) Read models / projections / results / wire types — data-only by
  design.** Built by a mapper or engine, consumed read-only, never
  persisted back as truth. No behavior is required; naming must make
  the class obvious.

Rules that follow from the classes:

- **Impossible states are unrepresentable.** If a state genuinely
  cannot occur, the constructor forbids it, full stop.
- **Legitimate states have exactly one owner of their meaning.** A state
  that looks irregular is usually legitimate. The model owns that
  meaning in one place — a normalizing factory or named behavior on the
  type — and no consumer re-decides it with a local `?:` fallback.
  Changing a producer to emit the normalized shape is part of the same
  chunk as its model, never a reason to keep the rule off the model.
- **Persistence entities stay rule-free.** Rules never live on entity
  classes in the data layer; the mapper seam (`toDomain()` /
  `toEntity()`) is construction only, not a home for logic.
- **Kotlin-rich, not Java-rich.** No private-field / getter-setter
  encapsulation theatre that fights data classes, persistence mapping,
  or Compose immutability: invariants in `init` / factories, behavior
  as member functions or extensions in the model's own file.
- **Use cases orchestrate.** They load, invoke domain behavior,
  persist, and publish. A validity rule or a state-meaning rule
  implemented in a use case, ViewModel, repository, or consumer
  fallback instead of on the owning domain model is a contract
  violation.

---

## 4. Presentation Contract

### 4.1 UI State

- Immutable `data class`. All updates via `copy()`.
- No mutable fields. No Android types inside UI state.
- One `StateFlow<UiState>` per ViewModel.

### 4.2 One-Shot Events

- Use `Channel<Event>(Channel.BUFFERED)` exposed as `Flow<Event>` via
  `receiveAsFlow()`.
- Consumed in `LaunchedEffect(Unit)` at the screen level.

### 4.3 Model Mapping

- Domain models MUST NOT be passed directly to composables. ViewModel
  maps domain → presentation.
- Presentation models (`*UiState`, `*UiEvent`, drafts) MUST NOT embed
  domain types; map at the ViewModel/mapper boundary.
- No repository calls inside composables.

### 4.4 ViewModel Dependencies & Flow Composition

- **ViewModels depend on use cases, not repository or port
  interfaces.** A ViewModel constructor injects `*UseCase` classes; it
  must not inject a `*Repository`/`*Port` interface or any data-layer
  type directly. Ports are consumed by use cases.
- **Mappers must not accept use-case parameters.** Mappers are pure
  domain → presentation transforms; ViewModels invoke use cases and
  pass derived values into mappers.
- **Use cases own multi-source `Flow` composition.** When a screen
  needs more than one source, add a use case that combines the streams
  into one domain read model; the ViewModel observes that single use
  case — it must not write its own multi-source `combine` over
  repositories.
- **Only transient/presentation state lives in the ViewModel** (unsaved
  drafts, reveal-errors flags, one-shot events). Durable facts are read
  through use cases.
- **Write paths go through use cases too** — never
  `repository.save(...)` from a ViewModel.

---

## 5. Navigation & Scaffold Architecture

Each screen owns its own `Scaffold`, `TopAppBar`, and `ViewModel`.
There is no global/root `Scaffold`.

```kotlin
// FeatureRoute.kt — wires ViewModel to Screen
@Composable
fun FeatureRoute(onBack: () -> Unit) {
    val viewModel: FeatureViewModel = hiltViewModel()
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    FeatureScreen(uiState = uiState, onBack = onBack)
}
```

- **No `DynamicTopBar`.** Each screen composes its own TopAppBar.
- **No `paddingValues` threading.** Each Scaffold consumes its own
  inner padding locally.
- Navigation callbacks are hoisted to the route level; the app shell
  owns the `NavHost` and passes lambdas.

---

## 6. Accessibility

- All interactive elements must have `contentDescription` + appropriate
  semantic `role`.
- Use `sp` units for text. System font scaling must be respected.
- TalkBack must work across all custom components, with a usable
  non-shortcut path through every screen.

---

## 7. Preview Requirements

Every composable file MUST include previews for:

- Light + Dark themes
- Empty state (where applicable)
- Populated / long content
- Loading state (where applicable)

*Constraint:* Preview composables must NOT reference Hilt or real
ViewModels. Use fake presentation models.
