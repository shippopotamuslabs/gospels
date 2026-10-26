# UNIT TESTING GOSPEL

The shared unit testing canon for Shippopotamus Android apps. It
prioritizes **Fakes over Mocks**, **JUnit 5**, and **Robolectric** for
high-fidelity JVM testing. Each repo's own unit testing contract
carries that repo's worked examples and gates; this gospel carries the
standards they all share.

---

## 1. Core Principles (ALWAYS / NEVER)

| Rule | Description |
| :--- | :--- |
| **ALWAYS** Prefer Fakes | Use real, lightweight in-memory implementations of owned application ports (shared `Fake*Port` fakes, `InMemoryDatabase`) instead of programming mock expectations. |
| **ALWAYS** JUnit 5 | Use `org.junit.jupiter.api` annotations (`@Test`, `@BeforeEach`, `@Nested`, `@DisplayName`). |
| **ALWAYS** State-Based Assertions | Assert on the *result* of an operation (e.g., the value in a StateFlow, a returned domain object, a DB row) rather than the *process* (e.g., "was this method called"). |
| **ALWAYS** Use Turbine | Use the Turbine library for testing Kotlin Flows and StateFlows to ensure event-by-event verification. |
| **ALWAYS** Main Dispatcher Rule/Extension | Use the shared `MainDispatcherRule` / `MainDispatcherExtension` to swap `Dispatchers.Main` for a `TestDispatcher`. |
| **ALWAYS** Copy Golden Patterns | Before inventing a test structure, open the matching golden reference (see §4). |
| **NEVER** Mock Android Classes | Do not mock `Context`, `SharedPreferences`, or `Bundle`. Use Robolectric's real implementations or appropriate fakes. |
| **NEVER** Mock Owned Ports | Do not `mockk` owned application ports the project owns. Use hand-written `Fake*Port` fakes. |
| **NEVER** `verify()` ViewModels | Do not use MockK `verify` / `coVerify` in `*ViewModelTest.kt` — assert UI state instead. |
| **NEVER** Use JUnit 4 in New Tests | Avoid `@Rule`, `@Test(expected=...)`, `@Before`, `@RunWith` in new tests. Migrated tests must be JUnit 5 compliant. |
| **NEVER** Mock Implementation Details | Avoid mocking private methods or internal helper classes. Test the public API of the unit. |
| **NEVER** Await Fire-and-Forget IO with `delay()` | Inside `runTest`, `delay(N)` advances *virtual* time (~0 wall-clock) and does **not** wait for `scope.launch(Dispatchers.IO) { … }`. Inject a `TestDispatcher` for the IO path and `advanceUntilIdle()`, or `join()` the background Jobs. See §3.3. |

---

## 2. Technical Stack

- **JUnit 5:** The engine for all local unit tests.
- **Hand-written fakes:** One `Fake*Port` per owned application port, with
  `seed` / `setShouldFail` / `clear` toggles. Reusable fakes live in
  the shared testing module; feature-local fakes only when single-use.
  A new outbound port ships its fake in the same change as the port.
- **MockK:** Use only when a Fake is impractical (third-party SDK
  boundaries, static Android APIs). Requires a file-level `@Suppress`
  with a one-line justification comment. Never for project-owned ports,
  never `verify` in ViewModel tests.
- **Turbine:** For all Flow/StateFlow assertions.
- **Robolectric:** For tests requiring Android framework APIs on the
  JVM (Room DAOs, DataStore, Context-dependent code).
- **StandardTestDispatcher:** Always use `StandardTestDispatcher` with
  `runTest` for coroutine testing, with explicit `advanceUntilIdle()`.

---

## 3. Patterns

### 3.1 ViewModel Testing (JUnit 5 + Turbine + Fakes)

Construct the ViewModel directly with fakes (or use cases backed by
fakes), register the Main dispatcher extension, and assert on the
`uiState` flow with Turbine: collect the initial state, invoke the
user action, then `awaitItem()` through the loading state to the final
state and assert on its content. State in, action, state out. No
`verify`, no interaction counting.

### 3.2 Fakes over Mocks

A mock programs expectations about how the code under test calls its
collaborators, so it breaks when the internal call pattern changes
even though behavior did not. A fake is a small real implementation:
an in-memory `MutableStateFlow` behind the port interface, plus
seed/failure toggles for the test to steer. Tests written against
fakes survive refactors that keep behavior; tests written against
mocks break on both.

### 3.3 Awaiting Fire-and-Forget Persistence (`runTest` + `Dispatchers.IO`)

An infrastructure adapter that updates an in-memory cache synchronously but
persists asynchronously — `scope.launch(Dispatchers.IO) { … }` —
**cannot** be awaited with a virtual-time poll inside `runTest`.

**Why it flakes:** `delay(N)` inside `runTest` advances *virtual*
time, not wall-clock time, so it completes in ~0 ms regardless of `N`.
The real `Dispatchers.IO` coroutine racing it may not have run yet, so
the assertion fires against stale state. Increasing `N` does not help.

Two correct shapes:

- **Option A (preferred): inject a `TestDispatcher` for the IO path.**
  Make the persistence dispatcher a constructor parameter, pass a
  `TestDispatcher` in tests, then `advanceUntilIdle()` —
  deterministic, no real threads.
- **Option B: `join()` the background Jobs.** When the class owns a
  real `CoroutineScope`, join its child coroutines before asserting,
  and assert both the cache and the persisted store.

Follow whichever option matches the class's construction; never bridge
real-dispatcher work with a `delay()` poll.

### 3.4 Test Placement by Layer

Where the test lives follows what the unit under test owns:

- **Domain policies** — plain JVM unit tests. No port fakes, no
  Robolectric: construct domain values, evaluate the policy, assert
  the decision. A policy test that needs a fake port has absorbed an
  application precondition — split it out (see
  `ARCHITECTURE-GOSPEL.md` §5).
- **Use cases** — plain JVM tests against `Fake*Port` fakes,
  asserting the sealed outcome: success, not-allowed (policy or
  precondition refused), or refused (fake steered to deny, standing in
  for the platform). No Robolectric, no real Android types.
- **Adapters** (infrastructure and `:app` port implementations) —
  Robolectric or instrumented tests asserting translation only: the
  right `Intent`, the right row, the right outcome branch, and error
  translation (platform exception → `Refused`). Never policy, never
  orchestration.
- **Entrypoints** — at most a translation test (event X triggers
  use-case call Y with input Z). If the entrypoint needs a decision
  test, the decision is misplaced — extract the use case (see
  `ARCHITECTURE-GOSPEL.md` §10–§11).
- **New outbound ports** — a new port ships its `Fake*Port` in the
  same change, living in the port-owning module's tests/fixtures
  (`:core:application` tests/fixtures) or the shared testing module
  (`:core:testing`).

---

## 4. Golden Test References

The Now in Android project's test files are the golden references for
these patterns: port-adapter fakes/tests, ViewModel JUnit 5 patterns,
Robolectric/Android logic, and the coroutines Main-dispatcher utility.
Each repo vendors or syncs its own copy (a `golden-test-examples/`
folder with a sync script) and adapts the patterns to its own
packages. Golden files are read-only references: adapt the pattern,
never copy-paste the imports into production modules.

---

## 5. Implementation Mandate

Any agent creating or modifying production logic **MUST**:

1. Read the repo's unit testing contract and the relevant golden
   example.
2. Include a matching unit test file in the `src/test` directory of
   the modified module, following these standards.
3. Run the narrowest affected test task during iteration, then broaden
   to the full unit test task for merge-sized or cross-module changes.

### 5.1 Self-check (anti reward-hacking)

After writing a test, break the production code under test (wrong
return value, skipped persistence). The test **must fail**. If it
still passes, the test is invalid — rewrite it before marking work
complete.
