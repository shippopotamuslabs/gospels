# RUST ARCHITECTURE GOSPEL

The architecture canon for Shippopotamus Rust projects. It states
the same intent as `ARCHITECTURE-GOSPEL.md` — Clean Architecture's
dependency rule — in the form Rust enforces natively. The two files
are siblings: read the one for the stack you are working in. Where
a repo's own contract and this gospel disagree on a general rule,
this gospel wins; where a repo contract records a local exception,
the exception stands and says so.

## 0. The Dependency Rule (Zero Tolerance)

Clean Architecture is the intent; the layer shape in `ARCHITECTURE-GOSPEL.md` §§1–14 is its
Kotlin/Android dialect. Rust enforces much of that intent in the
language — ownership, visibility, exhaustive matching, the type
system — so applying the Kotlin shape literally produces Rust that
fights itself: a trait for every class, `dyn Trait` everywhere,
anemic modules that only forward calls, and clones or broad locks
added to appease a structure the language never asked for.

A Shippopotamus Rust project therefore follows the intent through
Rust idioms. What survives translation, unchanged:

- **Dependencies point inward.** A pure core crate depends on no
  I/O, framework, transport, or storage crate. Edges depend on the
  core; the core depends on nothing outward.
- **Pure logic stays pure.** Decisions are functions of values.
  Time, randomness, filesystem, network, and process state enter
  at the edges and cross inward as plain values.
- **Effects live at the edges.** The outermost code performs the
  I/O; inner code states what it needs and returns what happened.
- **Substitution happens only at real boundaries.** A boundary
  exists where an implementation genuinely swaps — in production,
  in tests, or at runtime — not where a class diagram says one
  could.

## 1. Translation Table

| Clean / Kotlin concept (`ARCHITECTURE-GOSPEL.md` §§1–14) | Rust idiom | Drop |
|---|---|---|
| Rich aggregate / value object, invariants in `init` or a factory (`ARCHITECTURE-GOSPEL.md` §3.7) | Newtypes and parse-don't-validate: parsing constructs the checked type (`SnapshotRoot`, `ConfinedPath`); typestate encodes legal transitions, so an invalid transition does not compile | Runtime `validate()` checks and boolean validity flags re-decided by every consumer |
| Status and outcome modelled as sealed types (`ARCHITECTURE-GOSPEL.md` §8) | Algebraic data types: an `enum` per real state space, matched exhaustively (`ToolError`, `AgentAction`, `read_symbol` statuses) | Parallel booleans, string status fields, and `when`/`if` chains over values that should be variants |
| Outbound `*Port` interface for every external capability (`ARCHITECTURE-GOSPEL.md` §4) | A trait only at a genuine substitution boundary — a port in the true sense (`ModelTransport`, `ToolExecutor`) | A trait per service or repository class, named and shaped like the one implementation behind it |
| Infrastructure adapter implementing the port (`ARCHITECTURE-GOSPEL.md` §3.3) | Concrete types by default; generics and static dispatch where a boundary trait exists | `dyn Trait` as the default wiring, and object-safety contortions to keep it working |
| `*UseCase` orchestrator class per workflow (`ARCHITECTURE-GOSPEL.md` §3.2) | Plain functions and methods on the owning type; a module is the orchestration unit | A single-method struct per workflow, built only to hold injected ports |
| Hilt DI composition root (`ARCHITECTURE-GOSPEL.md` §3.6) | `main` (or the binary crate's entry point) constructs the concrete object graph by hand and passes it down | A DI container, annotation-driven wiring, or a service-locator module |
| Gradle module graph enforcing the layers (`ARCHITECTURE-GOSPEL.md` §6) | A Cargo workspace: crates are the hard architecture boundary, reinforced by `pub` / `pub(crate)` visibility inside a crate | Recreating domain / application / infrastructure package layers inside one crate as ceremony |
| `Repository` as infrastructure implementation vocabulary (`ARCHITECTURE-GOSPEL.md` §4) | A concrete store type; a trait only when the storage genuinely swaps or a test seam is load-bearing | A repository trait defined pre-emptively over a single store |
| Sealed outcome types distinguishing success / not-allowed / refused (`ARCHITECTURE-GOSPEL.md` §8) | Typed errors: `Result<T, E>` with an `enum` error per library crate, `From` conversions and `?` propagation; expected states stay enums, not errors | Stringly-typed errors, swallowed I/O errors, `unwrap()` as an escape hatch, and one universal error type for the whole workspace |
| DTOs kept off domain types, mapped at the boundary (`ARCHITECTURE-GOSPEL.md` §12.3) | Serde DTOs at the system edge only, with a stated unknown-field and versioning policy, translated into internal types via `From` / `TryFrom` | Serde-annotated core types, and external record shapes leaking inward as the domain model |
| Domain policy as a pure evaluator over domain values (`ARCHITECTURE-GOSPEL.md` §§3.1, 5) | A pure function over owned values and enums, with interior mutability — `Arc`, `Mutex`, `RwLock`, `OnceLock`, atomics — kept narrow and local to the type that owns the shared state | `&mut` threaded through the whole call graph to avoid a local lock, and locks held across unrelated work |

## 2. Rules That Follow

- **A trait earns its existence.** Name the substitution it enables
  (production vs test double, one transport vs another, runtime
  plugin) in the review or do not add the trait. Concrete first;
  abstract when the second implementation is real or scheduled.
- **Ownership is the API contract.** `self`, `&self`, and `&mut self`
  state who may do what next; consuming builders and `Drop` guards
  are design tools, not borrow-checker appeasement. A clone added to
  satisfy a layer boundary is a design smell to fix at the boundary.
- **Crates over layers.** When a boundary matters enough to police,
  it is a crate edge or a visibility wall, enforced by the compiler —
  never a comment, a naming convention, or a reviewer's memory.
- **Errors are part of the public surface.** Library crates expose
  typed errors a caller can match; only the binary / application
  edge converts errors into reports for humans.
- **This file is the whole Rust treatment.** Android-only material
  in `ARCHITECTURE-GOSPEL.md` (Compose, ViewModels, navigation,
  workers) does not apply to a Rust project, and none of it needs
  loading alongside this file. The dependency rule, purity, and
  boundary honesty stated here apply in full.
- **Reporting to a model is not swallowing.** Where a host's
  consumer is a language model working non-deterministically, an
  unreadable or unprocessable input is reported to the model as
  data — a soft failure it can react to — while the typed error
  still exists underneath for any caller who needs to match it.
  Swallowing is an error disappearing where a caller needed it;
  reporting is the error arriving in the form its reader can use.
