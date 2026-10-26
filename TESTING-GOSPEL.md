# TESTING GOSPEL

The shared testing canon for Shippopotamus software, on every stack.
It ranks three things in order: **stack-agnostic principles first**,
**Android specifics second**, **cross-stack mapping third**. The
default shape is the **test pyramid**; the default school is
**classical (Detroit)** state-based testing with **fakes over
mocks**; the binding rule is **the lowest level that can falsify the
behavior**.

Each repo's own testing contract carries that repo's worked examples
and gates (module paths, package names, golden copies, validation
commands); this gospel carries the standards they all share.

---

## 1. Test Levels and the Pyramid

### 1.1 Three levels

- **Small / unit.** Verifies a very small portion of the system — a
  method, a class, a policy, a use case against fakes. Fast,
  deterministic, narrow failure localization.
- **Medium / integration.** Checks the integration between two or
  more units across a **real seam**: real storage, real
  serialization, a real framework runtime, real IO scheduling. No
  full user journey.
- **Large / end-to-end.** Verifies a whole screen or user flow in a
  production-like environment. Highest fidelity, slowest, hardest to
  maintain, coarsest failure localization.

### 1.2 Tradeoffs

Lower levels are fast, reliable, and point at the broken unit when
they fail. Higher levels are closer to production (fidelity) but
slower, flakier, and harder to diagnose. That is the whole reason
for the pyramid: most tests small, fewer medium, fewest large.

Android's current documentation teaches exactly this qualitative
shape — "many small tests and relatively few big tests" — and notes
that hardware-dependent apps (media players, camera) may legitimately
change the shape. It also uses small / medium / big **size**
terminology based on scope and isolation, and allows a finer 5-layer
split (unit, component, feature, application, release-candidate)
for teams that want it, because the traditional three labels have no
precise definitions.

Where a test **runs** is orthogonal to its size: not all unit tests
are local, and not all end-to-end tests run on a device. A big local
test can run on the JVM through a simulator; a small instrumented
test can target a real database or framework integration across
devices.

### 1.3 The 70 / 20 / 10 orientation (not a gate)

As a rough starting shape, Google's testing literature suggests
**70% unit tests, 20% integration tests, 10% end-to-end tests** —
"as a good first guess", with "the exact mix will be different for
each team." Attribution matters: that ratio comes from the **Google
Testing Blog** (2015, "Just Say No to More End-to-End Tests"), **not**
from the current Android developer documentation, which deliberately
teaches the pyramid with **no numeric ratio**.

Treat 70 / 20 / 10 as an **orientation**, never a merge gate. The
binding rule is §2, not the percentages. A suite at 60 / 30 / 10
that puts every test at its lowest falsifying level is healthier
than a suite at exactly 70 / 20 / 10 that is not.

### 1.4 The inversion warning

An **ice-cream cone** (or inverted pyramid) — few unit tests, many
large end-to-end tests — is the canonical failure shape: slow
suites, flaky suites, failures that name a journey instead of a
unit. When the suite starts tasting like dessert, push tests down,
not up.

---

## 2. The Level-Escalation Decision Test

Apply this per test, not per suite:

1. **Start at the lowest level.** Write the test as a small / unit
   test first.
2. **Escalate to medium / integration only for seam risk**: real
   storage, real serialization, real IO scheduling or race behavior,
   framework lifecycle — things a fake cannot faithfully reproduce.
3. **Escalate to large / end-to-end only for critical journeys no
   lower level can falsify**: a full user flow whose failure modes
   live in the composition itself, on a production-like build.

If a lower level can falsify the behavior, the test belongs at that
lower level. Escalation needs a named seam risk or a critical
journey, not a feeling.

### 2.1 Mapping to architecture layers

(General form; §6 grounds it for Android. Layer names follow
`ARCHITECTURE-GOSPEL.md`: application ports, infrastructure
adapters, driving adapters.)

| Layer under test | Level | Why |
| :--- | :--- | :--- |
| Domain policy over domain values | Unit (small) | No ports, no IO — pure meaning. |
| Use case against `Fake*Port` fakes | Unit (small) | Fakes are deterministic and in-process; no real seam is crossed. Classified honestly: small, not integration, even when it wires several collaborators. |
| Adapter translation against a real seam (real database, real framework runtime, instrumented device API) | Medium (integration) | The seam is real; translation and error mapping are what is being falsified. |
| Screen flow (state in, action, state out through real UI machinery) | Medium to large | A real composition is exercised but not a full journey. |
| Full journey on a production-like build | Large (end-to-end) | Only for critical flows; nothing lower falsifies composition failures. |

**Driving adapters** (entrypoints: receivers, activities mapping a
tap) get at most a translation test — event X triggers use-case
call Y with input Z. If the entrypoint needs a decision test, the
decision is misplaced: extract the use case (see
`ARCHITECTURE-GOSPEL.md` §10–§11).

**New outbound ports** ship their `Fake*Port` in the same change,
living in the port-owning module's tests/fixtures or the shared
testing module — so the use-case level always has a fake to stand
on.

---

## 3. Schools: Classical Is Canon

### 3.1 The two schools, fairly stated

- **Classical (Detroit / Chicago) TDD**, in the lineage of Beck's
  *Test-Driven Development By Example*: verify **state**, use **real
  objects wherever possible**, and reach for a test double only
  where the real thing is awkward — slow, nondeterministic, or
  side-effecting.
- **Mockist (London) TDD**, in the lineage of Freeman & Pryce's
  *Growing Object-Oriented Software Guided by Tests*: verify
  **interactions**, mock collaborators with interesting behavior,
  and drive design outside-in from the object's protocol. Its real
  benefit is as a **design discipline**: thinking hard about
  collaborator protocols early.

### 3.2 Verdict: classical is canon

This gospel standardizes on the classical school: state-based
assertions over real code paths, with fakes standing in at owned,
expensive, or external boundaries.

Mockist-by-default is rejected as canon for two reasons:

1. **Brittleness under behavior-preserving refactor.** Tests that
   encode call patterns break when the internal call pattern
   changes even though behavior did not.
2. **Mock drift.** A mock is a private theory of the collaborator's
   behavior. Nothing keeps that theory honest when the real
   collaborator evolves; a fake with working behavior at least
   executes.

(Corrections recorded so the canon does not caricature its rival:
mockists mock collaborators *with interesting behavior*, not
literally everything — value objects stay real. And classical
fallback is usually a *stub*, not necessarily a fake; the
criterion is awkwardness, not externality.)

### 3.3 Narrow exceptions

Mocks and spies are acceptable only at these edges, with a
one-line justification, preferring spies and fakes first:

- An **unowned third-party or static platform boundary** that is
  impractical to fake (third-party SDK, static Android API).
- A **pure interaction assertion** with no observable state to
  check — e.g. an analytics dispatch — where there is nothing to
  assert except that the call happened.

Preference ladder, strongest first: **real implementation > fake >
stub / spy > mock.**

---

## 4. Test Doubles

Definitions follow Fowler's test-double vocabulary:

- **Dummy:** passed around but never used. Fills parameter lists.
- **Stub:** gives canned answers to calls made during the test.
- **Fake:** a working implementation with a production-unsuitable
  shortcut (e.g. an in-memory test database). Behaves like the real
  thing within the test's scope.
- **Spy:** a stub that also records how it was called.
- **Mock:** pre-programmed with expectations forming a
  specification of the calls it expects to receive. Fails the test
  on unexpected calls; verified at the end.

### 4.1 Rules

- **Never mock types you own — especially application `*Port`s.**
  Write a hand-written `Fake*Port` and ship it in the same change
  as the port. One fake per owned port, with seed / failure /
  clear toggles.
- **Never mock the unit's internals or value objects.** Test the
  public API of the unit; private methods and internal helpers are
  exercised through it.
- **Fakes must be faithful where behavior matters.** A fake is a
  theory of the real implementation too — keep it contract-faithful,
  test the fake itself where its behavior is load-bearing, and share
  contract tests between the fake and the real adapter where that
  is applicable. An unfaithful fake is mock drift with better
  manners.
- **Mocks only at unowned boundaries, with justification.**
  Third-party SDKs and static platform APIs a fake cannot reach;
  file-level suppression plus a one-line reason (see §6.1 for the
  Android form of this rule). Prefer wrapping the foreign type
  behind a port or adapter you own, then faking that owned seam in
  consumers' tests; reserve a mock for the foreign boundary itself
  inside the adapter's own translation test when a fake cannot reach it.
- **Prefer state verification.** Assert on outcomes (returned
  objects, state-holder contents, stored rows), and reserve
  interaction verification for the narrow exceptions in §3.3.

---

## 5. General Best Practices

Only what the sources or the prior canon ground — kept concise,
not a textbook dump:

- **Assert state and outcomes, not interactions.** The result of
  the operation (a returned domain object, a state-holder value, a
  stored row), not "was this method called".
- **Unit tests are deterministic: no real time, no real network.**
  Fake the clock, advance virtual time explicitly, keep IO
  in-process. A test that depends on wall-clock scheduling is a
  flaky test wearing a passing disguise.
- **Test behavior through the public API, one behavior per test.**
  Name the test for the behavior (`condition → expected result`),
  so the failure names the broken promise.
- **Flaky tests are defects.** Quarantine and fix; never hide a
  flake behind retries or longer sleeps. A retry that passes is
  evidence destroyed, not confidence gained.

---

## 6. Android Application

Everything above binds every stack. This section grounds it for
Android, and preserves every rule of the prior Android canon,
re-homed.

### 6.1 Core principles (ALWAYS / NEVER)

| Rule | Description |
| :--- | :--- |
| **ALWAYS** Prefer Fakes | Use real, lightweight in-memory implementations of owned application ports (shared `Fake*Port` fakes, `InMemoryDatabase`) instead of programming mock expectations. |
| **ALWAYS** JUnit 5 | Use `org.junit.jupiter.api` annotations (`@Test`, `@BeforeEach`, `@Nested`, `@DisplayName`). |
| **ALWAYS** State-Based Assertions | Assert on the *result* of an operation (e.g., the value in a StateFlow, a returned domain object, a DB row) rather than the *process* (e.g., "was this method called"). |
| **ALWAYS** Use Turbine | Use the Turbine library for testing Kotlin Flows and StateFlows to ensure event-by-event verification. |
| **ALWAYS** Main Dispatcher Rule/Extension | Use the shared `MainDispatcherRule` / `MainDispatcherExtension` to swap `Dispatchers.Main` for a `TestDispatcher`. |
| **ALWAYS** Copy Golden Patterns | Before inventing a test structure, open the matching golden reference (see §6.4). |
| **NEVER** Mock Android Classes | Do not mock `Context`, `SharedPreferences`, or `Bundle`. Use Robolectric's real implementations or appropriate fakes. |
| **NEVER** Mock Owned Ports | Do not `mockk` owned application ports the project owns. Use hand-written `Fake*Port` fakes. |
| **NEVER** `verify()` ViewModels | Do not use MockK `verify` / `coVerify` in `*ViewModelTest.kt` — assert UI state instead. |
| **NEVER** Use JUnit 4 in New Tests | Avoid `@Rule`, `@Test(expected=...)`, `@Before`, `@RunWith` in new tests. Migrated tests must be JUnit 5 compliant. |
| **NEVER** Mock Implementation Details | Avoid mocking private methods or internal helper classes. Test the public API of the unit. |
| **NEVER** Await Fire-and-Forget IO with `delay()` | Inside `runTest`, `delay(N)` advances *virtual* time (~0 wall-clock) and does **not** wait for `scope.launch(Dispatchers.IO) { … }`. Inject a `TestDispatcher` for the IO path and `advanceUntilIdle()`, or `join()` the background Jobs. See §6.3.3. |

### 6.2 Technical stack

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
  JVM (Room DAOs, DataStore, Context-dependent code). Robolectric is
  a JVM simulator for high-fidelity **unit and UI-behavior** tests —
  not a device replacement and not end-to-end: prefer refactoring
  code to be testable without shadows where possible, keep screenshot
  use to cases where pixel accuracy is not critical, and still cover
  system UI, WebView, and unsupported APIs on device.
- **StandardTestDispatcher:** Always use `StandardTestDispatcher` with
  `runTest` for coroutine testing, with explicit `advanceUntilIdle()`.

### 6.3 Patterns

#### 6.3.1 ViewModel testing (JUnit 5 + Turbine + Fakes)

Construct the ViewModel directly with fakes (or use cases backed by
fakes), register the Main dispatcher extension, and assert on the
`uiState` flow with Turbine: collect the initial state, invoke the
user action, then `awaitItem()` through the loading state to the final
state and assert on its content. State in, action, state out. No
`verify`, no interaction counting.

#### 6.3.2 Fakes over mocks

A mock programs expectations about how the code under test calls its
collaborators, so it breaks when the internal call pattern changes
even though behavior did not. A fake is a small real implementation:
an in-memory `MutableStateFlow` behind the port interface, plus
seed/failure toggles for the test to steer. Tests written against
fakes survive refactors that keep behavior; tests written against
mocks break on both.

#### 6.3.3 Awaiting fire-and-forget persistence (`runTest` + `Dispatchers.IO`)

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

#### 6.3.4 Test placement by layer

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

In level terms (§2.1): policies and use cases are unit / small;
adapters against real Room, DataStore, or device APIs are medium /
integration; Compose screen flows (§6.3.5) are medium to large;
full journeys on device (§6.3.6) are large.

#### 6.3.5 Compose UI tests

Screen flows use `createComposeRule()` with semantic matchers,
asserting user-observable state and behavior (state in, action,
state out). This is the Android form of testing the way the
software is used: match on what the user sees, not on
implementation internals.

#### 6.3.6 Instrumented tests and full journeys

Instrumented tests run on physical or emulated devices with an
injected test app — usually UI tests. Reserve them for what the
JVM cannot falsify: real framework behavior across devices (small
instrumented tests, e.g. SQLite / framework integration) and full
critical user journeys on production-like builds (large). Device
time is the most expensive time in the pyramid; spend it only
where §2 says to escalate.

### 6.4 Golden test references

The Now in Android project's test files are the golden references for
these patterns: port-adapter fakes/tests, ViewModel JUnit 5 patterns,
Robolectric/Android logic, and the coroutines Main-dispatcher utility.
Each repo vendors or syncs its own copy (a `golden-test-examples/`
folder with a sync script) and adapts the patterns to its own
packages. Golden files are read-only references: adapt the pattern,
never copy-paste the imports into production modules.

### 6.5 Implementation mandate

Any agent creating or modifying production logic **MUST**:

1. Read the repo's unit testing contract and the relevant golden
   example.
2. Include a matching unit test file in the `src/test` directory of
   the modified module, following these standards.
3. Run the narrowest affected test task during iteration, then broaden
   to the full unit test task for merge-sized or cross-module changes.

#### 6.5.1 Self-check (anti reward-hacking)

After writing a test, break the production code under test (wrong
return value, skipped persistence). The test **must fail**. If it
still passes, the test is invalid — rewrite it before marking work
complete.

---

## 7. Cross-Stack Mapping

The rules in §§1–5 are pattern-neutral: lowest falsifying level,
classical school, real > fake > stub/spy > mock, no owned-type
mocks, deterministic units, state over interaction. What changes per
stack is only which tools play each role. Tool names below are
**examples in parentheses, never mandates** — outside Android (§6),
this gospel mandates no tool.

**TypeScript note:** TypeScript is a language, not a separate
stack. The React row covers TypeScript-in-React; the same rules and
the same runner examples apply to TypeScript-on-Node services. It
gets no duplicate row.

| Stack | Unit / small | Medium / integration | Large / end-to-end | Boundary-fake pattern |
| :--- | :--- | :--- | :--- | :--- |
| React (+ TypeScript) | Component tests asserting user-observable behavior (render + user-event over a real DOM, e.g. jsdom) — biased *up* toward how the software is used, not component instances | HTTP faked at the network layer with app code unmodified (e.g. MSW); module seams mocked only at edges, never for owned logic | Real-browser journeys asserting user-visible behavior with isolated state per test (e.g. Playwright); network faked only at the transport edge (e.g. route interception); never depend on implementation internals | Classical fake preference is strongest at the network edge: fake the transport, run the real code |
| Go | Table-driven unit tests against small interfaces with hand-written fake implementations ("accept interfaces, return structs"); no mock framework needed | Real HTTP clients/servers on loopback (e.g. `httptest` server and response recorder) at the HTTP boundary | End-to-end client tests against real servers on loopback where the composition itself is what is being falsified | Interface-seam fakes are the idiom; function-patching is non-idiomatic |
| Python | Fixtures as the setup/teardown and dependency-injection graph; real collaborators by default | Environment and network seams steered per test with automatic undo (e.g. `monkeypatch`), full-stack exercise where feasible | Critical user journeys against production-like environments | Patch at the env/network seams, including own functions only where the seam cannot be easily tested another way — never as the default collaborator strategy |

Per-stack cautions, honestly labeled:

- Component-test restraint (do not mock owned modules, do not
  assert on implementation details) comes from the **testing
  philosophy layer** (e.g. Testing Library's guiding principles),
  not from the runner: module-mocking mechanics (e.g. `vi.mock`)
  document no such restraint themselves.
- Python's fixture/patch tooling **allows** patching own functions;
  the gospel's classical preference still applies — patch at
  seams, prefer real collaborators and fakes elsewhere.
- Go has no module-mocking concept at all; the language's interface
  shape already enforces the seam discipline.

---

## 8. Sources / References

- [Android testing strategy pyramid](https://developer.android.com/training/testing/fundamentals/strategies) — many-small / few-big qualitative pyramid, 5-layer option, no numeric ratio.
- [Android testing fundamentals](https://developer.android.com/training/testing/fundamentals) — small / medium / big sizes; local vs instrumented as the primary distinction, orthogonal to size.
- [Robolectric on Android developers](https://developer.android.com/training/testing/local-tests/robolectric) — JVM simulator for unit and UI tests; not a device replacement; prefer refactoring without shadows.
- [Android test doubles](https://developer.android.com/training/testing/fundamentals/test-doubles) — fake/mock/stub/dummy/spy/shadow definitions; fakes preferred over stubs; fakes or mocks preferred over spies; hermetic tests and DI.
- [What to test on Android](https://developer.android.com/training/testing/fundamentals/what-to-test) — persistence and network doubles; activities/fragments hold no business logic.
- [Just Say No to More End-to-End Tests (Google Testing Blog, 2015)](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html) — origin of the 70 / 20 / 10 first-guess split.
- [Fowler: Test Double](https://martinfowler.com/bliki/TestDouble.html) — dummy / stub / fake / spy / mock taxonomy.
- [Fowler: Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) — classical vs mockist; real objects where possible; mocks for collaborators with interesting behavior.
- [Fowler: Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html) — many low-level tests over few broad-stack GUI tests.
- [TotT: Fake Your Way to Better Tests](https://testing.googleblog.com/2013/06/testing-on-toilet-fake-your-way-to.html) — a fake is a lightweight implementation unsuitable for production.
- [TotT: Don't Overuse Mocks](https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html) — prefer hermetic servers and fakes over mocks where possible.
- [TotT: Don't Mock Types You Don't Own](https://testing.googleblog.com/2020/07/testing-on-toilet-dont-mock-types-you.html) — Google's advice to wrap unowned types behind an owned seam (this gospel fakes that owned seam per s4.1 rather than mocking it).
- [TotT: Know Your Test Doubles](https://testing.googleblog.com/2013/07/testing-on-toilet-know-your-test-doubles.html) — double definitions; prefer real or fake execution over verifying state-changing calls.
- [Software Engineering at Google, Ch. 13: Test Doubles](https://abseil.io/resources/swe-book/html/ch13.html) — real implementation first, fake second; avoid interaction testing where possible; mocks legitimate for hard-to-trigger paths.
- [London vs Detroit school naming](https://www.tricentis.com/learn/test-driven-development) — mockist/London vs classicist/Detroit-Chicago mapping; lineages tied to Freeman & Pryce and Beck.
- [Testing Library guiding principles](https://testing-library.com/docs/guiding-principles) — tests resembling software use; DOM nodes over instances.
- [Vitest mocking guide](https://vitest.dev/guide/mocking) — `vi.mock` mechanics, hoisting, automock strategies.
- [Vitest module mocking](https://vitest.dev/guide/mocking/modules) — factory / no-factory / `__mocks__` strategies.
- [Jest manual mocks](https://jestjs.io/docs/manual-mocks) — user-module vs Node-module mocking.
- [Vitest request mocking (MSW recommendation)](https://vitest.dev/guide/mocking/requests) — mock network via Mock Service Worker, not module patches.
- [MSW best practices](https://mswjs.io/guides/best-practices/) — network-boundary fakes; describe the network you want.
- [Playwright best practices](https://playwright.dev/docs/best-practices) — user-visible behavior; isolated tests; don't test third-party dependencies.
- [Playwright network mocking](https://playwright.dev/docs/mock) — route interception and fulfillment at the transport edge.
- [Go httptest package](https://pkg.go.dev/net/http/httptest) — loopback servers and response recorders from stdlib.
- [Go testing talk (2014)](https://go.dev/talks/2014/testing.slide) — testing HTTP clients and servers with `httptest`.
- [Go tutorial: add a test](https://go.dev/doc/tutorial/add-a-test) — table-driven test idiom.
- [pytest monkeypatch](https://docs.pytest.org/en/stable/how-to/monkeypatch.html) — seam patching with automatic undo.
- [pytest fixtures explained](https://docs.pytest.org/en/stable/explanation/fixtures.html) — fixtures as named setup/teardown and DI.
- [pytest fixtures how-to](https://docs.pytest.org/en/stable/how-to/fixtures.html) — fixture mechanics.

