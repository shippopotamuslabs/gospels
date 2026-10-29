# CODE STYLE GOSPEL

The shared Kotlin and Compose code style canon for Shippopotamus
Android apps: idioms, naming, formatting, and stability rules. Each
repo's own code style contract holds that repo's example tables and
enforcement lines (see "Local material" at the end).

---

## Strict Boundaries

| Always | Never |
| :--- | :--- |
| Use `val` over `var` and immutable collections by default. | Use `!!` (non-null assert) without `requireNotNull()`. |
| Accept `modifier: Modifier = Modifier` as the **last** parameter. | Apply the passed `modifier` to an inner element instead of the outermost layout. |
| Use `kotlinx.collections.immutable` in UI state classes. | Scatter `@Stable` or `@Immutable` preemptively without profiling. |
| Prefer method references (`::onModeChange`) for callbacks. | Create new lambda instances inside `LazyColumn` or `LazyRow` item blocks. |
| Use `stateIn` with `WhileSubscribed(5_000)` for ViewModel state. | Manually `.collect` and `.update` state inside `viewModelScope.launch`. |
| Use `collectAsStateWithLifecycle()` in Compose. | Use `collectAsState()`, `GlobalScope`, or `runBlocking`. |
| Use `derivedStateOf` for state that depends on other state. | Perform heavy calculations or filtering directly in the Composable body. |
| Prefer Slot APIs (`content: @Composable () -> Unit`) for layout flexibility. | Pass 10+ specific data parameters into a single "God Composable." |
| Hoist business state to the ViewModel. | Hoist ephemeral UI state (focus, animation) out of Compose unnecessarily. |
| Use `CompositionLocal` only for truly cross-cutting themes/configs. | Use `CompositionLocal` to pass data that should be explicitly passed via parameters. |
| Comment the "why" when logic is non-obvious. | Comment the "what" or leave `// TODO` comments in merged code. |
| Use `state.update { }` to update `MutableStateFlow` | Use `state.value =` |

## 1. Kotlin Conventions

- Kotlin 2.x with K2 compiler.
- Explicit return types on all public functions and properties.
- `val` over `var`. Immutable collections by default.
- `when` over `if-else` chains for 3+ branches.
- No `!!` (non-null assert). Use `requireNotNull()` with a message, or handle nulls explicitly.
- Extension functions for repeated utility logic; keep them in focused files, not scattered.
- Named arguments on any call with 3+ parameters or where meaning is ambiguous.

## 2. Naming

| Element | Convention |
|---------|-----------|
| Package | lowercase dot-separated |
| Class / Object | PascalCase |
| Function | camelCase, verb-first |
| Composable | PascalCase (noun) |
| Property | camelCase |
| Constant | UPPER_SNAKE_CASE |
| Private backing flow | `_uiState` pattern: `private val _uiState = MutableStateFlow(...)` |
| Resource IDs | snake_case |
| Application / outbound ports | `*Port` suffix on the interface; capability not mechanism; no `I-` prefix; no mechanism words (`Foreground`, `Context`, `Intent`) in name or signature; domain/application types only |
| Port implementations | `Android*` / `DataStore*` / `Room*` + capability (e.g. `AndroidBubbleServiceStarter`); `Repository` allowed only as such an implementation-class name (e.g. `Room*Repository` implementing a `*Port`), never as interface vocabulary |
| Use cases | `*UseCase` suffix; live in Application (see `ARCHITECTURE-GOSPEL.md` §3.2) |
| Domain policies | `*Policy` suffix; pure functions, never `*Port` |
| Domain services | Plain capability name, never `*Port` (a domain interface implemented inside Domain is a domain service, not a port) |
| Fakes | `Fake*Port` hand-written fakes, one per owned application port |

## 3. File Organization

- One public top-level composable per file.
- Helper composables and preview composables go below the main public composable as `private`.

### 3.1 Package Layout Mirrors the Layers

Packages follow the layer/module map in `ARCHITECTURE-GOSPEL.md` §6:
`domain` (models, `*Policy`, domain services), `application`
(`*UseCase`, `*Port`, outcomes), `infrastructure` (port adapters,
persistence as a subpackage — there is no sibling `data` layer),
`presentation` (screens, ViewModels, mappers), `entrypoints`
(receivers, services, workers, activities), `di` (composition root).
A type lives in the package of the layer that owns it; a use case
anywhere but `application`, a port anywhere but `application`, or a
persistence type outside `infrastructure` is a misplacement.

## 4. Compose-Specific Rules

### 4.1 Stability
- UI state classes: `data class` (stable by default when all fields are stable types).
- If a class contains `List`, `Map`, or other mutable collection types, either:
  - Use `kotlinx.collections.immutable` (`ImmutableList`, `PersistentList`), OR
  - Annotate with `@Immutable` / `@Stable` if you guarantee no mutation.
- **Do not scatter `@Stable` annotations preemptively.** Only add when profiling confirms unnecessary recomposition from that specific type.

### 4.2 Modifier Conventions
- Every composable that emits layout accepts `modifier: Modifier = Modifier` as its **last** parameter.
- Apply `modifier` to the outermost layout element.
- Modifier order matters: `padding` before `background` clips differently than the reverse. Follow the logical layering.

### 4.3 State Hoisting
- Composables that manage text input: accept `value: String` + `onValueChange: (String) -> Unit`.
- Internal ephemeral state (`remember { mutableStateOf(...) }`) is fine for dropdown expanded, focus, animation state.
- Business state always hoisted to ViewModel.

### 4.4 Lambdas
- Prefer method references (`viewModel::onModeChange`) over inline lambdas when passing to child composables.
- Avoid creating new lambda instances in Lazy item blocks (causes recomposition). Extract to `remember` or use stable references.

### 4.5 Rigor Protocols
- **Visual Purity (Canvas):** If a progress bar artifact appears, use explicit overrides like `drawStopIndicator = {}` and `gapSize = 0.dp`. Use `StrokeCap.Butt` with a parent `Modifier.clip` to ensure clean rounding.
- **Performance Deferral:** High-frequency state (like download progress percentages) must be passed to leaf composables as a lambda `() -> Float`. This ensures the root screen does not recompose on every byte update.

### 4.6 Plausible Composition
- All `LazyColumn` and `LazyRow` components must use stable keys to maximize the efficiency of the `RecordingApplier` during pausable frames.
- Ensure state reads are deferred using lambda-based modifiers (e.g., `Modifier.offset { ... }` or `drawBehind`) to minimize recomposition.

## 5. Coroutines & Flow

- `viewModelScope.launch` for ViewModel work.
- `collectAsStateWithLifecycle()` (not `collectAsState()`) for StateFlow in composables.
- `LaunchedEffect(Unit)` for one-shot event collection from `Channel`/`Flow`.
- Never use `GlobalScope`.
- Never use `runBlocking` in production code.

## 6. KDoc & Comments

- KDoc on public classes and non-obvious public functions only.
- Inline comments only where the "why" is non-obvious. Never comment the "what" when the code is self-explanatory.
- No `TODO` comments in merged code — use the issue tracker.
- File-level comments only for contracts or architectural rationale.

## 7. Formatting

- 4-space indent (Kotlin standard).
- Max line length: 120 characters.
- Trailing commas on multi-line parameter lists and `when` branches.
- Blank line between functions. No multiple consecutive blank lines.
- Import ordering: Kotlin stdlib → Android/Compose → Third-party → Project. Let the IDE sort.

## 8. Testing Conventions

- Unit tests: JUnit 5 + Turbine for Flow testing + **hand-written fakes** for project-owned ports (see `TESTING-GOSPEL.md` §6). MockK only for third-party SDK boundaries with `@Suppress` + justification; no `verify` in ViewModel tests.
- Compose UI tests: `createComposeRule()` with semantic matchers.
- Test names: `fun methodName_condition_expectedResult()` or backtick descriptive names.
- Fakes for previews live alongside models. Fakes for tests live in the shared testing module or the module `test/` source set (single-use only).

### 8.1 Test Coverage Discipline (Agent Mandate)

- Whenever the task **adds** new production code (new classes, public functions, significant logic in domain, infrastructure, ViewModels, port adapters, mappers, use cases, etc.) **or modifies** existing code, the agent **MUST**:
  - Write new unit tests (or extend existing ones) in the matching `src/test/kotlin` source set that cover the new/modified behavior, main paths, edge cases, and error conditions.

  - Use JUnit 5 + Turbine (for `StateFlow`/`Flow`) + hand-written fakes as defined above and in `TESTING-GOSPEL.md` §6.
- For pure UI/Compose changes that affect behavior or interactions: also add or update Compose UI tests using `createComposeRule()` and semantic matchers.
- Trivial styling-only or preview-only changes are exempt from new tests, but the full test suite must still pass.
- Before declaring work complete, the agent must have written/updated the tests **and** executed them successfully.

## 9. Worker Exception Handling

| Always | Never |
|--------|-------|
| Explicitly catch and rethrow `CancellationException`. | Catch `Exception` generically without re-throwing cancellation. |
| Return `Result.retry()` on transient network/IO failures. | Return `Result.failure()` for recoverable quota timeouts. |

The general failure rules (classification, handling, emission) are §11; this table is the worker-specific shape.

## 10. Partial Success: Diagnostics as Values

When a computation can degrade without failing — it skips a step, drops an input, falls back to a default — the degradation is reported as data, not logged and not swallowed. Below the boundary there is no other sanctioned channel: throwing kills work that can still succeed, and silent skipping makes the defect unobservable.

- **Shape.** The computation returns an outcome: its value plus a list of diagnostics. Diagnostics are a sealed type owned by the feature area, defined in the domain layer next to the computation's models (ARCHITECTURE-GOSPEL §3.1 counts diagnostics among the domain's contents). The outcome rides in the operation's normal transport (`Result<Outcome>` when the computation can also fail outright).
- **Diagnostics are data.** Each diagnostic carries what was skipped or degraded, why, and the identifying context needed to act on it (the entity, the position, the offending input). No logging, no ports, no I/O below the boundary; a diagnostic is constructed, never emitted.
- **The boundary drains once.** The caller at the application/feature boundary iterates the diagnostics on success and logs each exactly once through its existing logging wiring. The partial-success rule mirrors the failure rule: log once, at the boundary, from data.
- **Diagnostics never reach the UI.** They are operational signal, not user copy. If the user needs to know, that is product state (next rule).
- **Diagnostics vs state.** When a degradation is itself meaningful to the product, something the user or another computation should react to, model it in the returned state instead. Diagnostics cover degradations the system tolerates at runtime but must be able to see in logs. When in doubt, state is the richer claim; diagnostics are the default for purely operational loss.
- **Tests assert diagnostics as values.** A degradation path is covered by asserting the exact diagnostics on the returned outcome. Verifying a logger mock is not a substitute.
- **Adoption is incremental.** New degradation paths use the pattern from the start. Existing silent paths adopt it when next touched under their own ticket; there is no retrofit sweep.

**Worked example.** Stratum's progression snapshot (STR-250): the sealed `ProgressionDiagnostic` in the domain records each history baseline the load math had to skip and why; the snapshot use case returns the diagnostics on its outcome; the session runtime at the feature boundary logs each exactly once. Stratum's code style contract carries the file-level detail as local material.

## 11. Failure Classification: Expected vs Exceptional

Every failure in a Shippopotamus codebase belongs to exactly one class, decided by
what the failure *is*, never by how it happens to be carried. The class decides
everything downstream: the mechanism, the owner, the telemetry home, and whether
anything is allowed to alert on it.

- **A normal answer is not a failure.** A predicate question (does this key exist, is
  this value valid) returns a plain value: a boolean, an `Option` / nullable, an empty
  collection. Both answers are equally valid outcomes, so there is nothing to
  classify.
- **An expected failure is the system doing its job.** A rejection, a failed
  precondition, malformed input, an expected provider condition: routine outcomes of
  correct code meeting the real world. Expected failures are handled gracefully
  inside the app, return an intentional handled result, and are recorded as
  structured expected-outcome telemetry. They raise no alert and crash nothing.
- **An exceptional failure is the system broken.** A programmer error, a broken
  invariant, an impossible state, an unexpected provider or system failure.
  Exceptional failures are alert-worthy and surface as 500-class failures. "500-class"
  names each surface's exceptional expression: an HTTP 500 where the surface has
  responses; on screens, workers, and tasks, the boundary containment below.
- **A partial success is neither.** A computation that degrades without failing
  follows §10: diagnostics as values, drained once at the boundary.

### Classification by semantic owner

Classify a failure by the semantic owner of the failed condition:

- Malformed transport input belongs to **presentation**.
- A semantic business rejection uses a **domain-owned** failure type: a domain
  outcome branch in Result-pattern code, or a specific domain exception subtype where
  exceptions are the mechanism.
- A use-case precondition or coordination failure may use an **application-owned**
  type.
- **Infrastructure** translates expected provider conditions at its boundary into
  the explicit inward result or exception the port requires. Provider errors never
  leak inward (ARCHITECTURE-GOSPEL §3.3); translation keeps the original error
  attached as the cause, so the evidence survives the type change.

Do not force every routine failure into the domain, and do not let a failure drift
into a class it does not belong to. An internal argument-contract violation is a
programmer error and stays exceptional, even when the call originated in
presentation.

### Mechanism

- **The Result pattern is the default** in every Shippopotamus stack, stated as each
  language's expression of the same rule: Kotlin sealed outcomes and `Result`, Rust
  `Result<T, E>`, Go multiple return values `(T, error)`, Python and TypeScript
  Result-style types. Where a language or library surface does not support it well,
  use the idiomatic mechanism for that surface and name the fallback in the repo's
  contract.
- **Classification follows semantics, not mechanism.** An expected outcome stays
  expected however it is carried; wrapping a business rejection in an exception type
  does not make it exceptional.
- **No failure mechanism as control flow.** A throw steers nothing routine. Branching
  on a typed Result branch is the mechanism working as designed: visible in the
  signature, local to the caller.
- **Expected failures stay explicit and typed at every layer they cross**: a branch
  of the result or outcome type (ARCHITECTURE-GOSPEL §8), never a side channel.
  Presentation preserves intentional handled outcomes (a refusal shown to the user, a
  validation result, a non-500 response) instead of collapsing them into generic
  errors.
- **Exceptional failures use the most specific type available** where exceptions are
  the mechanism: the most specific built-in or custom exception type that fits; a
  generic base exception only for an unexpected failure when no meaningful specific
  type exists.
- **Cancellation is not a failure.** Cancellation propagates as cancellation: no
  error result, no error log, no alert, no failure telemetry (§9 covers the worker
  shape).

### Handling

- **Handle each failure exactly once.** At each layer a failure is handled there
  (decision made, outcome produced) or propagated. Never both.
- **Handling is terminal and changes the type.** Handling consumes the failure and
  returns a non-error value: a response, a UI state, an outcome object. Code that has
  handled a failure has nothing left to propagate.
- **Boundary containment for the exceptional case.** Each boundary (request handler,
  screen, task) aborts the operation, records the full detail internally, and returns
  a generic user-facing message with a correlation reference. Internals (stack
  traces, paths, provider messages) never reach the user. Where state cannot be
  trusted, stopping the operation, and if needed the process under a supervisor,
  beats continuing in an unknown state.
- **The user always gets a friendly error**, for expected and exceptional failures
  alike. The classes differ in telemetry and alerting, never in courtesy.

### Emission: one home per class

Classification also decides the telemetry store. Each class has exactly one home, so
nothing is sent twice and no two series can disagree.

- **Exception records** (fatals, ANRs, handled exceptional failures recorded by the
  boundary) live in **Crashlytics**, the crash sensor: automatic capture,
  symbolication, grouping, breadcrumbs. Crashlytics data reaches Grafana through
  Crashlytics' native BigQuery export, queried via Grafana's BigQuery data source.
- **Outcome telemetry** (successes included; rates need denominators), traces,
  metrics, and operational facts live in the **OpenTelemetry stream** to Grafana.
- **The single designed exception** is a bare OTel counter marking that an
  exceptional failure occurred, for real-time rate alerting. The record itself stays
  in Crashlytics; the counter's rate alert is the real-time form of the
  exceptional-class alert, not a second page on the same condition.
- **A telemetry router per app** (infrastructure layer) owns the class-to-store
  mapping. It is the only code referencing either the Crashlytics SDK or the OTel
  exporter, gate-enforced. Its API is typed by telemetry kind with no destination
  parameter: callers declare what a fact is, the router decides where it goes.
  Layer-scoped facades give each outer layer only the emit methods legal for it
  (presentation emits outcomes, never exception records). Scrubbing, consent, and
  sampling are applied inside the router. Emission facades are app-shell and platform
  mechanisms owned by the boundary: not outbound ports, not use cases (this is the
  stated reconciliation with ARCHITECTURE-GOSPEL §3.4 and §3.5, not an override).
- **Correlation attributes** are shared across both stores: session, app version,
  hashed install id, trace id, stamped into Crashlytics custom keys and OTel
  attributes alike, so the two stores pivot into each other.
- **Alerting consolidates into Grafana.** Classification decides alert eligibility;
  volume decides paging. Expected failures never alert. Exceptional failures are
  recorded as errors and are alert-eligible; paging fires on actionable rates and
  burn, not on a single occurrence. Once an app's Grafana wiring is proven, all of
  its alerting consolidates into Grafana and its Crashlytics alert rules retire; the
  Crashlytics SDK remains the capture mechanism and stops being an alerting surface.
  During wiring, existing Crashlytics alerts stay live so coverage never gaps.
- **Export authentication is a per-app choice, made explicitly.** Where a repo's
  contract forbids embedding write tokens in the shipped artifact, export goes
  through an authenticated collector (the Firebase Auth / App Check callable shape).
  An existing embedded-key arrangement in one app is that app's gated legacy pattern,
  never a default another app inherits silently.

### Enforcement

The gospels are working rules for agents, not essays. Each rule below states its
verification method; a rule an agent cannot be audited against does not ship.

- **E1. The propagation path has no logging capability.** Domain and application
  define no logging port, inject no logger, and import no logging or telemetry
  library. A logging port in either layer is a layer violation on its face. Logging
  lives in the outer layers only: presentation, infrastructure, platform
  entrypoints, the app shell. Infrastructure logs its own translation work;
  presentation and entrypoints log what they handle; the boundary logs what reaches
  it. *Verification:* module and package dependency checks show no logging or
  telemetry dependency in domain and application; a repo-wide search for logger
  types, logging ports, and log calls in those source sets returns zero hits.
- **E2. Exactly one handler per boundary, and reaching it means unhandled.** Each
  boundary names one handler that logs and renders exceptional failures. Code below
  the boundary never logs an error it propagates; its legal moves are annotate (wrap
  with context, cause preserved), translate at a port boundary, recover fully, or
  propagate. *Verification:* search catch blocks and error branches outside the
  boundary for log calls; each hit is a violation unless the same block fully handles
  the failure and returns an outcome.
- **E3. Handling is terminal and changes the type.** *Verification:* functions that
  log an error do not also return or throw that same error; functions that return an
  error do not log it. Review and lint check the pairing.
- **E4. Catch blocks have three legal shapes.** (1) **Recover:** produce the handled
  outcome and return it. (2) **Translate:** wrap in the owning layer's type, original
  attached as cause, and propagate. (3) **Boundary render:** log once with full
  detail, return the generic user-facing result. Any other shape (empty catch,
  catch-and-continue, catch-log-rethrow, catch-and-return-default that hides a
  failure) is a violation to fix or to flag with a ticket.
- **E5. Lints and architecture tests carry the rules.** Each consuming repo wires the
  checks its stack supports: forbidden-import and module-dependency rules for E1,
  static analysis for the E2/E3/E4 shapes, must-use and unchecked-error linters where
  the language provides them (Rust `Result`, Go error checks). The gospel states the
  rule and the kind of gate; each repo's contract names its exact tools and commands,
  per the Local material split. *Verification:* the PR that adopts this canon in a
  repo names the gates it added and shows them running.
- **E6. Agent conduct.** When you create a failure, classify it (normal answer /
  expected / exceptional) in the same change; an unclassified new error type or throw
  site is unfinished work. When you touch a catch block or an error branch, bring it
  to an E4 shape in the same change when the fix is small; when it is not small, file
  the ticket and leave a `// DEBT:` reference per ARCHITECTURE-GOSPEL §11. Never add
  a logger, a logging port, or a telemetry call to the domain or application layer to
  solve a local visibility problem: surface the fact in the outcome type and let the
  handling layer record it.

### Worked mapping

| Case | Owner | Mechanism | Handled outcome | Telemetry |
|------|-------|-----------|-----------|-----------|
| Malformed input | Presentation | Validation result at the boundary | User-facing correction, non-500 response | Expected-outcome record in OTel |
| Business rejection | Domain | Domain outcome branch / specific domain exception subtype | The refusal, shown or returned as designed | Expected-outcome record in OTel |
| Precondition failure | Application | Application-owned failure type on the outcome | Operation declined with its reason as state | Expected-outcome record in OTel |
| Expected provider condition | Infrastructure | Translated at the port boundary, cause attached | The port's explicit inward result | Expected-outcome record in OTel |
| Unexpected failure | Boundary | Most specific exception type; boundary containment | Generic user message with correlation reference | Exception record in Crashlytics (BigQuery feed to Grafana) + bare OTel occurrence counter |

Cross-references: outcome types are ARCHITECTURE-GOSPEL §8; adapter translation is
§3.3; partial success is §10 of this gospel; worker retry/failure results are §9 of
this gospel.

## Local material

Example tables with a repo's own package names, class names, and
resource IDs live in that repo's code style contract, next to the code
they describe. Enforcement lines also live there: the repo's Detekt
rule set and rule names, its Linear team and ticket references, and
its exact validation commands. This gospel carries the rules; the
repo contract carries the repo's names and gates.
