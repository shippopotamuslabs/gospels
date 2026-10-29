# ARCHITECTURE GOSPEL — CLEAN ARCHITECTURE

The shared Clean Architecture canon for Shippopotamus Android apps.
Rust projects follow the same intent — above all the dependency
rule — in the form stated by `RUST-ARCHITECTURE-GOSPEL.md`; when
the stack is Rust, read that file instead of this one.
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
Dependencies point inward toward the domain layer; infrastructure and
presentation code adapts framework, storage, platform, and UI concerns
around application outbound ports, with pure domain logic at the center.

**The agent MUST NEVER:**

- Write, propose, or allow any code that violates the dependency rule.
- Leak Android, Compose, Hilt, Room, WorkManager, or any other
  framework/SDK implementation types through domain or application
  public APIs.
- Bypass layer boundaries, forbidden imports, or layer responsibilities.
- Use any other architectural pattern (MVVM-only, monolithic, etc.).
- Put business logic in the app shell or in composables.
- Pass domain models directly to composables.
- Pass `NavController` into composables.

**Violation = immediate contract breach.**
Reply with exactly: "Contract violation detected in [file]. Request explicit 'Contract Waiver' from user."
Then stop. Do not generate code until waived.

All rules below are non-negotiable implementations of Clean Architecture.

**Rust scope note:** Rust codebases are governed by
`RUST-ARCHITECTURE-GOSPEL.md`, which states this same dependency
rule in Rust form. A Rust codebase that follows that file is
compliant, not in breach, and no Contract Waiver is required. A
Rust codebase that drops the dependency rule itself — effects in
the pure core, external types leaking across a crate boundary — is
in breach exactly as a Kotlin codebase would be.

---

## Strict Boundaries

| Always | Never |
|--------|-------|
| Route navigation via pure callbacks hoisted to the route level. | Pass `NavController` into Composables. |
| Expose a single `StateFlow<UiState>` from ViewModels. | Expose mutable flows or Android framework types from ViewModels. |
| Ensure each screen owns its own `Scaffold` and `TopAppBar`. | Thread `paddingValues` down the UI tree from a root-level Scaffold. |
| Consume one-shot events via `Channel<Event>` and `LaunchedEffect`. | Use `SharedFlow` for one-shot UI events (can cause dropped events). |
| Use stable keys on all `LazyColumn` and `LazyRow` items. | Import infrastructure, presentation, app-shell, feature, UI, or platform-bridge code into the domain layer. |
| Enforce Clean Architecture dependency inversion at module boundaries. | Allow reverse dependencies into the domain layer. |
| Keep domain APIs as domain models, domain services/policies, and test fixtures. | Expose Android/Compose/Hilt/Room or other framework types from domain or application APIs. |
| Make Workers thin driving adapters: translate the work request into a single use-case call. | Embed orchestration (multi-port reads, policy checks, conditional effects) in a `Worker`. |
| Route background progress and retry/idempotency mechanics through the use case and its ports. | Scatter unmanaged work requests, singleton callbacks, or static listeners for background-to-UI updates. |

Work-request mechanics (`ExistingWorkPolicy`, `setProgress`, backoff,
byte-offset resumption) are platform details owned by the driving
adapter and the repo contract — see that repo's architecture contract
and [CODE-STYLE-GOSPEL.md](CODE-STYLE-GOSPEL.md) §9. They never change
the layer rule: the Worker decides nothing.

---

## 1. Dependency Direction (Strict)

- Dependencies point inward. Domain depends on nothing else in the app:
  no app, feature, infrastructure, UI, platform-entrypoint,
  presentation, or platform-bridge modules, and no Android framework
  types in its public APIs.
- Infrastructure, UI, platform-entrypoint, presentation, and feature
  code may depend inward. The application layer sits between domain
  and the outer layers and depends on domain only.
- Infrastructure modules implement application outbound ports and never
  import the app shell or presentation code.
- Feature modules depend on domain, application (use cases),
  and shared UI, and never on infrastructure directly;
  port-to-adapter bindings are made by app-shell DI. Feature
  modules never depend on other feature modules; cross-feature
  navigation is wired in the app shell via callbacks and shared routes.
- The DI package/module is the composition root: the only place ports
  meet implementations.
- External libraries are consumed behind an application outbound port;
  library types do not leak into domain, application, or
  presentation APIs.

**No reverse dependencies into the domain layer are allowed.** Each
repo's architecture contract carries its exact module graph, and the
graph is enforced by Gradle wherever the module split exists.

Port ownership is defined in §4: outbound ports are defined in the
application layer, implemented by infrastructure adapters. No domain
port survives — see §4.

---

## 2. Forbidden Imports (Principles)

| Layer | Must NOT import |
|-------|-----------------|
| Domain | Application, infrastructure, platform-entrypoint, presentation, app shell, feature, UI, or platform-bridge code; Compose; Hilt annotations on domain types; Android framework types in public APIs; library implementation classes |
| Application / use cases | Infrastructure, platform-entrypoint, presentation; Compose; Android framework types; library implementation classes |
| Infrastructure | Compose UI; presentation; platform entrypoints; the DI package; anything in the app shell |
| Presentation | Infrastructure classes and library implementation types — go through use cases only (ports are called only from use cases); domain types in composables and `UiState` (ViewModels may receive domain values back from use cases and map them — see §3.4) |
| Platform entrypoints / services | Outbound ports, infrastructure types, or domain policies directly — depend on use cases only (see §3.5) |
| App shell / DI | Nothing is forbidden structurally — it wires everything — but it must contain no business rules, no orchestration, and no policy (see §3.6) |
| Feature | Other feature modules |

Reusable UI code lives with the presentation layer or the shared UI
module; screens are wired by the app shell.

---

## 3. Layer Responsibilities

### 3.1 Domain — pure business logic

- **Purpose:** Decide what things mean and what is allowed, as pure
  functions of domain values. No I/O, no time, no randomness, no state
  publishing, no trigger knowledge.
- **Owns:** Models, value objects, aggregates, domain services,
  evaluators/policies (pure function domain → domain/decision),
  diagnostics, and test fixtures.
- **May depend on:** Nothing else in the app. Pure Kotlin/JVM,
  framework-free.
- **Must never know:** Use cases, ports, `Flow` collection, triggers
  (boot, buttons, workers), Android/persistence/UI types, I/O,
  storage, network, time, or randomness.
- **Canonical contents:** `*Policy` evaluators, domain services (no
  `*Port` suffix — a domain interface implemented inside Domain is a
  domain service/policy, never a port), aggregates with enforced
  invariants, value objects.
- **Misplacement anti-example:** A `ShouldAutoStartPolicy` that reads a
  DataStore flag or takes an `isBootCompleted: Boolean` trigger
  parameter. The DataStore read is an application port load; the
  trigger fact is an application precondition (§5). The policy takes
  domain values only.

### 3.2 Application — the orchestrator

- **Purpose:** Run workflows. Every use case / interactor in the
  codebase lives here. A use case loads state via ports, applies
  application preconditions, invokes domain, calls ports for effects,
  and returns an application outcome.
- **Owns:** Use cases (application services, `*UseCase`), outbound
  `*Port` interfaces, application preconditions, sealed outcome types.
- **May depend on:** Domain only. Pure Kotlin/JVM.
- **Must never know:** Android, storage, network, files, third-party
  SDKs, Compose, platform trigger types, or any mechanism
  word (`Foreground`, `Context`, `Intent`) in port names/signatures.
- **Canonical contents:** `Report*UseCase`, `Observe*UseCase`,
  `Reconcile*UseCase`; `*Port` interfaces named for capability in
  application vocabulary (e.g. `BubbleServiceStarterPort.start(origin)`);
  `Fake*Port` test fixtures in this module's tests/fixtures or the
  shared testing module.
- **Misplacement anti-example:** A use case that builds an `Intent` or
  calls `ContextCompat.startForegroundService`. That is translation —
  it belongs in the infrastructure adapter implementing the port.

### 3.3 Infrastructure — translation to the outside world

- **Purpose:** Translate between application port calls and the outside
  world. The only layer that knows Android, storage, network, files,
  or third-party SDKs.
- **Owns:** Adapter implementations of application outbound ports
  (`AndroidBubbleServiceStarter`, `DataStore*`, `Room*Repository` —
  `Repository` is allowed only as such an implementation-class name,
  never as interface vocabulary), entity ↔ domain mapping, error
  translation.
- **May depend on:** Application (port interfaces) and Domain (values
  crossing the port). Framework/SDK libraries.
- **Must never know:** Use-case internals, presentation, entrypoints,
  app shell. No Android/IO/Room/SDK exception type crosses a port —
  refusals become application outcomes (§8).
- **Canonical contents:** Port adapters, Room DAOs/entities + mappers,
  DataStore readers/writers, SDK wrappers behind ports.
- **Misplacement anti-example:** A Room DAO query that encodes "user
  may start" policy (e.g. `WHERE setupFinished = 1 AND ...`). That
  precondition belongs in the use case; the DAO answers dumb storage
  questions only.

### 3.4 Presentation — render and report

- **Purpose:** Render state and report user intent. Map domain values
  received back from use cases into presentation models at the
  ViewModel/mapper boundary.
- **Owns:** Compose UI, ViewModels, mappers (pure domain → presentation
  transforms), `UiState`, one-shot events.
- **May depend on:** Use cases only for decisions and data. A ViewModel
  may receive domain values back from a use case and map them.
- **Must never know:** Outbound ports, repositories, infrastructure
  types, domain policies. Composables and `UiState` never carry domain
  types. No port reads, no multi-source `combine` over ports, no
  effect decisions.
- **Canonical contents:** `*Screen` composables, `*ViewModel`
  (injects `*UseCase` only), `*UiState`, pure mappers.
- **Misplacement anti-example:** A ViewModel that injects an onboarding
  port and a presence port, combines both `Flow`s, applies a start
  policy, and calls a starter port. Reading N ports + policy +
  conditional effect is orchestration — it is a use case however small
  (§10 shows the shape).

### 3.5 Platform entrypoints — driving adapters

- **Purpose:** Translate a platform event into one application input,
  call the use case, and report the outcome. Broadcast receivers,
  activities, accessibility services, foreground/overlay services,
  workers, tile services, content providers are all driving adapters.
- **Owns:** Event → application-input mapping, the single use-case
  call, outcome reporting.
- **May depend on:** Use cases only.
- **Must never know:** Outbound ports, infrastructure types,
  domain policies. Never injects an outbound port. Never reads two
  sources and applies policy. Never decides an effect.
- **Canonical contents:** Thin `BroadcastReceiver`, `Worker`,
  `Service`, `Activity` shells delegating to `*UseCase`.
- **Misplacement anti-example:** A `BroadcastReceiver` injecting an
  onboarding port and a resident-presence port, reading both, applying
  a domain start policy, then calling `ContextCompat` itself. That is
  orchestration in the entrypoint — extract a use case (§10, §11).

### 3.6 App shell / DI — composition root, wiring only

- **Purpose:** Wire everything together. Application class, navigation
  wiring, DI composition, app resources, manifest component
  declarations.
- **Owns:** Module bindings (port → adapter), `NavHost`, app-level
  resources.
- **May depend on:** Everything structurally — that is its job.
- **Must never know:** Business rules, orchestration, policy. No
  business logic, no conditional effects, no policy checks — only
  wiring and composition.
- **Canonical contents:** Hilt modules, `NavHost`, `Application` class.
- **Misplacement anti-example:** A DI module whose `@Provides` method
  checks setup state and conditionally binds a no-op starter. That is
  a precondition decision — it belongs in a use case, not in wiring.

### 3.7 Domain model policy: rich aggregates, honest projections

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
  classes in the infrastructure layer; the mapper seam (`toDomain()` /
  `toEntity()`) is construction only, not a home for logic.
- **Kotlin-rich, not Java-rich.** No private-field / getter-setter
  encapsulation theatre that fights data classes, persistence mapping,
  or Compose immutability: invariants in `init` / factories, behavior
  as member functions or extensions in the model's own file.
- **Use cases orchestrate.** They load, invoke domain behavior,
  persist, and publish. A validity rule or a state-meaning rule
  implemented in a use case, ViewModel, infrastructure adapter, or
  consumer fallback instead of on the owning domain model is a contract
  violation.

---

## 4. Port Ownership (Outbound Ports Live in Application, Full Stop)

**Rule:** Outbound ports are defined in the Application layer, named
for capability in application vocabulary. No domain port survives.

- **Decision test.** If the implementation would live outside Domain,
  or needs I/O, platform, storage, time, randomness, or
  state-publishing → it is an application port. If it is a pure
  function domain → domain/decision → it is a policy/domain service,
  not a port.
- **Naming:** `*Port` suffix, capability not mechanism, no `I-`
  prefix, no mechanism words (`Foreground`, `Context`, `Intent`) in
  the port name or signature. Port signatures carry domain/application
  types only.
- **Implementation:** Infrastructure adapters implement the ports
  (`AndroidBubbleServiceStarter : BubbleServiceStarterPort`) and hold
  all framework calls (`ContextCompat.startForegroundService`) plus
  error translation inside the adapter.
- **Retired vocabulary:** `Repository` is retired as interface
  vocabulary. Application defines `*Port` interfaces; `Repository`
  survives only as an infrastructure implementation-class name (e.g. a
  `Room*Repository` implementing a `*Port`). The implementor chooses
  the mechanism (repository, API, file, SDK); Application never knows.

```kotlin
// Application — port, capability vocabulary, framework-free
interface BubbleServiceStarterPort {
    fun start(origin: StartOrigin): StartOutcome
}

// Infrastructure — adapter, holds ALL framework calls
class AndroidBubbleServiceStarter @Inject constructor(
    @ApplicationContext private val context: Context,
) : BubbleServiceStarterPort {
    override fun start(origin: StartOrigin): StartOutcome = try {
        ContextCompat.startForegroundService(context, bubbleIntent(context, origin))
        StartOutcome.Started
    } catch (e: IllegalStateException) {
        StartOutcome.Refused.PlatformDenied
    }
}
```

---

## 5. Domain Invariant vs Application Precondition

A **domain invariant** is what a domain value means — true regardless
of who asks or what trigger fired (e.g. `autoStartEnabled &&
!userStopped` inside a start-eligibility policy). An **application
precondition** is a current-situation fact gating a workflow: setup
finished, signed in, permission granted, model downloaded, already
running, origin of the request.

- Preconditions are checked by the use case **before** invoking the
  domain policy — never smuggled in as extra policy parameters.
- A policy that takes trigger facts (`isBootCompleted`), permission
  flags, or "already running" state as parameters has absorbed an
  application precondition. Split it: the use case gates, the policy
  decides meaning.
- Tests follow the split: policy tests assert meaning over domain
  values; use-case tests assert gating over fake-port state (see
  [TESTING-GOSPEL.md](TESTING-GOSPEL.md)).

---

## 6. Module Map Template

```
:core:domain         — pure JVM. Domain models, policies, domain services.
:core:application    — pure JVM. Use cases, outbound *Port interfaces,
                       outcomes, Fake*Port fixtures (tests/fixtures).
:core:infrastructure — Android/storage/network/SDK. Implements the
                       application ports. Persistence is a subset of
                       this module, not a sibling.
:app                 — shell + DI + presentation + entrypoints.
:core:testing        — (optional) shared fakes, Turbine/Main-dispatcher
                       utilities, golden-pattern helpers.
```

`:core:data` has no independent existence. Where that name appears in
older repos it is a legacy label for what is now a persistence package
inside `:core:infrastructure`, and those repos refactor toward this
map. Do not present `:core:data` as a canonical equal.

**Enforcement honesty:** the `:core:domain` / `:core:application` /
`:core:infrastructure` module edges are Gradle-enforced (no forbidden
module dependency compiles). Inside `:app`, the
presentation / entrypoint / app-shell separation is package discipline
only — enforced by review and the forbidden-imports table (§2) —
unless the repo splits feature modules, in which case each repo's
architecture contract records the exact enforced graph.

---

## 7. Adapter Placement

- A **framework-only adapter** (needs only `Context`, Room, DataStore,
  SDK types) lives in `:core:infrastructure` and is bound via DI.
- An **adapter referencing an `:app` type** (a `Service`/`Activity` /
  manifest-declared component class) lives in `:app` and is bound via
  DI there. The module dependency forbids the reverse.
- **Named exception, not technique:** flattening a component name to a
  `String` to dodge the module dependency is a recorded exception for
  that call site, with reason and owner in the repo contract — never a
  pattern to copy. The default is: the adapter lives where its types
  live.

---

## 8. Outcomes Rule

Use cases and ports return sealed outcomes that distinguish three
shapes: **success** (the effect happened), **not-allowed** (policy or
precondition refused — no platform call was attempted), and
**refused** (infrastructure/platform denied — the call was attempted
and the world said no).

```kotlin
sealed interface StartOutcome {
    data object Started : StartOutcome
    data object NotAllowed : StartOutcome   // policy/precondition refused
    sealed interface Refused : StartOutcome {
        data object PlatformDenied : Refused // infrastructure/platform denied
    }
}
```

Android exception types never appear in an application signature. The
adapter catches platform exceptions and maps them to the `Refused`
branch; preconditions and policy refusals map to `NotAllowed` before
any platform call.

---

## 9. Desired State vs Actual State (Persisted Intent vs Runtime Actuals)

Persisted intent (what the user asked to be true — e.g. "bubble
enabled") and runtime actuals (what is currently running/published)
are different facts with different owners, not copies to keep in sync.

- **Convergence is an application use case** (e.g.
  `ReconcileBubbleUseCase`): load intent via ports, load actuals via
  ports, invoke the domain policy, call effect ports.
- **Entrypoints signal trigger/origin only** — boot completed, package
  replaced, resume, toggle. They never reconcile. The use case
  reconciles given the origin.
- **Intent controls render intent; status surfaces render actuals.**
  Never render one and write the other.
- **State-holder / publish ports** are application ports whose
  implementations are in-memory singletons in infrastructure — or in
  `:app` when the holder references an `:app` type. Holders carry
  runtime actuals only, never persisted intent. Services publish state
  **only** through a use case (e.g. `ReportPresenceUseCase` recording
  via a port). UI reads state **only** through an observe use case
  returning the `Flow` once — not per-emission port calls from the
  ViewModel.

---

## 10. Worked Example — Boot Receiver (Bad vs Good)

Pieces and where they live:

| Piece | File / module |
|-------|---------------|
| `BootReceiver` (driving adapter) | `:app`, entrypoints package |
| `ReconcileBubbleUseCase` | `:core:application`, use-case package |
| Setup precondition + `ShouldAutoStartPolicy` (domain) | `:core:domain` |
| `BubbleServiceStarterPort` | `:core:application`, port package |
| `AndroidBubbleServiceStarter` (adapter) | `:core:infrastructure` (or `:app` if it references an `:app` `Service` type — §7) |
| In-app user start from activity | Same port, same use case, activity maps the tap to the call |

**Bad** — orchestration embedded in the entrypoint:

```kotlin
// :app — DO NOT DO THIS
class BootReceiver : BroadcastReceiver() {
    @Inject lateinit var onboardingPort: OnboardingPort   // NEVER: port in entrypoint
    @Inject lateinit var presencePort: ResidentPresencePort
    override fun onReceive(context: Context, intent: Intent) {
        val onboarded = onboardingPort.isOnboarded().first()   // orchestration
        val present = presencePort.presence().first()
        if (onboarded && ShouldAutoStartPolicy.evaluate(present)) {
            ContextCompat.startForegroundService(context, ...) // framework in entrypoint
        }
    }
}
```

Three violations: entrypoint injects outbound ports, reads two sources
and applies policy (orchestration → a use case however small), and
makes the framework call itself.

**Good** — receiver maps the event; the use case orchestrates:

```kotlin
// :app — receiver translates BOOT_COMPLETED / MY_PACKAGE_REPLACED to one call
class BootReceiver : BroadcastReceiver() {
    @Inject lateinit var reconcile: ReconcileBubbleUseCase
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action in setOf(Intent.ACTION_BOOT_COMPLETED, Intent.ACTION_MY_PACKAGE_REPLACED)) {
            goAsync { reconcile(StartOrigin.Boot) }
        }
    }
}

// :core:application — use case: precondition, policy, effect port
class ReconcileBubbleUseCase @Inject constructor(
    private val setup: SetupStatePort,
    private val policy: ShouldAutoStartPolicy,
    private val starter: BubbleServiceStarterPort,
) {
    suspend operator fun invoke(origin: StartOrigin): StartOutcome {
        if (!setup.isSetupFinished()) return StartOutcome.NotAllowed
        if (!policy.evaluate(loadEligibility())) return StartOutcome.NotAllowed
        return starter.start(origin)
    }
}
```

The same port serves the in-app user start: the activity maps the tap
to `reconcile(StartOrigin.User)` and renders the returned outcome. One
orchestration, two triggers, zero duplicated policy.

---

## 11. When-You-Touch-It Rule

Modifying an entrypoint that embeds orchestration (port reads, policy
application, conditional effects) requires extracting the use case in
the same change when the extraction is small and behavior-preserving.
When it is not small, file a ticket naming the debt (component,
orchestration found, owning team) and link it from a `// DEBT:` comment
at the call site. There are no silent sanctioned exceptions: a real
exception is written into the repo's architecture contract with reason
and owner. "Too small to matter" is never a reason — §10 shows the
extraction is small by construction.

---

## 12. Presentation Contract

### 12.1 UI State

- Immutable `data class`. All updates via `copy()`.
- No mutable fields. No Android types inside UI state.
- One `StateFlow<UiState>` per ViewModel.

### 12.2 One-Shot Events

- Use `Channel<Event>(Channel.BUFFERED)` exposed as `Flow<Event>` via
  `receiveAsFlow()`.
- Consumed in `LaunchedEffect(Unit)` at the screen level.

### 12.3 Model Mapping

- Domain models MUST NOT be passed directly to composables. ViewModel
  maps domain → presentation.
- Presentation models (`*UiState`, `*UiEvent`, drafts) MUST NOT embed
  domain types; map at the ViewModel/mapper boundary.
- No port or infrastructure calls inside composables.

### 12.4 Driving-Adapter Dependencies & Flow Composition

- **All driving adapters depend on use cases only** — ViewModels,
  activities, broadcast receivers, accessibility services,
  foreground/overlay services, workers, tile services, content
  providers. A driving-adapter constructor injects `*UseCase` classes;
  it must not inject an outbound port or any infrastructure type
  directly. Outbound ports are defined in Application and called only
  from use cases.
- **Mappers must not accept use-case parameters.** Mappers are pure
  domain → presentation transforms; ViewModels invoke use cases and
  pass derived values into mappers.
- **Use cases own multi-source `Flow` composition.** When a screen
  needs more than one source, add a use case that combines the streams
  into one domain read model; the ViewModel observes that single use
  case — it must not write its own multi-source `combine` over ports.
- **Only transient/presentation state lives in the ViewModel** (unsaved
  drafts, reveal-errors flags, one-shot events). Durable facts are read
  through use cases.
- **Write paths go through use cases too** — never a direct port call
  from a driving adapter.

---

## 13. Navigation & Scaffold Architecture

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

## 14. Accessibility & Previews (Layer Statements)

- **Accessibility is a presentation-layer obligation.** Interactive
  elements expose `contentDescription` + semantic `role`; text uses
  `sp` and respects system font scaling; TalkBack reaches every screen
  through a usable non-shortcut path. Presentation owns the widgets;
  no other layer changes to satisfy it. (Full checklist: each repo's
  UI contract.)
- **Previews are a presentation-layer obligation.** Every composable
  file ships light + dark, empty, populated/long-content, and loading
  previews; preview composables reference only fake presentation
  models — never Hilt or real ViewModels. (Full checklist: each repo's
  UI contract.)
- **Accessibility services are driving adapters.** An accessibility
  service translates platform events into use-case calls per §3.5 —
  the a11y API shapes the event mapping, never the orchestration.

---

## 15. Rust Dialect — Moved

The Rust dialect lives in its own file,
`RUST-ARCHITECTURE-GOSPEL.md`. Read it when the stack is Rust;
nothing in §§0–14 needs loading alongside it.
