# The Doodle Engine — Embedding and Instrumentation API

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-07-09

This document specifies the **Doodle engine**: the embeddable core interpreter,
the interface through which a host injects capabilities (turtle graphics, input,
files, …), and the facilities through which a host **drives and observes**
execution (stepping, source position, call-stack inspection, breakpoints). It is
an adjacent specification to the [Doodle Language Specification](language.md),
referred to below as **L** (so **L§8.7** means §8.7 of the language spec). It is
derived from the design discussion archived in
[`../Claude-Doodle language discussion.md`](<../Claude-Doodle language discussion.md>).

Where the discussion or the language spec left a question open, or where writing
a precise rule forced a choice, this document makes a call and records it in
[Appendix B](#appendix-b-decisions-and-open-issues).

---

## 1. Introduction

The Doodle language core has no input/output, graphics, clock, or randomness
(L§14): every capability a program uses is supplied by its host. The engine is
the boundary. It runs Doodle programs deterministically and lets a host (a) add
capabilities, (b) resolve modules, and (c) drive execution one observable step at
a time. The interactive environment — REPL, canvas, debugger, animation — is not
a separate layer but simply the richest host: it uses the same API a game engine
or a command-line runner would.

### 1.1 Scope

This specification defines, abstractly:

- the engine's lifecycle and instance model (§3);
- the value model at the host boundary, including foreign values (§4);
- foreign functions and native modules — the "hooks" (§5);
- module resolution (§6);
- the **resumable drive loop** that runs, suspends, pauses, and resumes execution
  (§7);
- **observation and instrumentation** — source position, the tail-call-aware call
  stack, value inspection, breakpoints, stepping (§8);
- errors and protected execution (§9);
- control, limits, and the concurrency model (§10);
- determinism and the recordable boundary that enables replay/time-travel (§11);
- the shape of the reference bindings (§12) and the relationship to the standard
  library (§13).

### 1.2 What this specification does not cover

- **Language semantics** — see L. This document assumes the value model (L§4),
  scope and dynamic parameters (L§5), procedures/functions/blocks and proper tail
  calls (L§8), modules (L§11), errors (L§12), reflection (L§13), and the
  execution model (L§14).
- **The standard library** and the specific list of *platform primitives* it
  requires — a separate specification. This document defines the *mechanism* by
  which such primitives are provided (§5, §13), not the list.
- **A concrete interactive environment.** This document specifies the facilities
  an environment uses; it does not specify the REPL/canvas/debugger UI, animation
  policy, or `turtle_speed` semantics (those belong to the environment and the
  turtle library).
- **A frozen ABI or a specific header.** The API is specified as a *semantic
  contract*; §12 describes reference bindings (a C ABI and a WASM/JS binding), but
  the concrete header and JS surface are separate artifacts.

### 1.3 Design principles

1. **The core is pure and deterministic.** Given a program and a fixed sequence
   of results from the host, the engine's execution is fully reproducible. All
   nondeterminism enters through the host boundary (§11).
2. **Capabilities come only from the host.** The engine has no built-in effects.
   `print`, `forward`, `read_line`, files, time, and randomness are all
   host-provided foreign functions or capabilities.
3. **Observation is native, not a mode.** Stepping, pausing, and inspection are
   the ordinary way a host drives the engine, not a special instrumented build.
   "Execution is visible by default" is achieved by making the debug interface
   the engine's normal interface.
4. **One mechanism for running, blocking, and debugging.** The engine is a
   resumable state machine (§7); a blocking capability, a breakpoint, and a step
   boundary are all just reasons the drive loop returns control to the host.
5. **The boundary is recordable.** Because all nondeterminism crosses the
   boundary, a host that records what crosses it can replay execution exactly
   (§11).

### 1.4 Terminology and conformance

- **Engine** — an implementation of this contract. **Instance** (or *runtime*) —
  one live engine with its own isolated heap and execution state.
- **Host** — the program embedding the engine. **Environment** — a host that
  provides an interactive experience (the reference environment is the Doodle
  IDE).
- **Capability** — an effect the host performs on the program's behalf. **Foreign
  function** — a host-supplied callable exposed to Doodle. **Foreign value** — a
  host object exposed to Doodle as an opaque value.
- **Handle** — a host-held reference to a Doodle value (§4.2). **Drive** — advance
  execution. **Outcome** — why a drive returned (§7.2). **Safe point** — a place
  where the engine may stop and be fully inspected and resumed (§7.4). **Frame** —
  an activation on the call stack (§8.2).

The key words **must**, **must not**, **should**, and **may** carry their
ordinary normative sense. An engine is **conforming** if it implements this
contract with the semantics given here. A host is unconstrained by this document
except that a host claiming to be a *platform host* for the standard library must
additionally provide the platform primitives that specification requires (§13).

---

## 2. Architecture

Three layers:

```
  ┌─────────────────────────────────────────────┐
  │  Interactive environment (REPL, canvas,      │   the richest host
  │  debugger, animation) — an ordinary host     │
  ├─────────────────────────────────────────────┤
  │  Host: provides capabilities, resolves       │   uses the embedding API
  │  modules, drives & observes execution        │
  ├─────────────────────────────────────────────┤
  │  Engine (core interpreter): pure, deterministic,   the embedding API
  │  resumable; owns parsing, evaluation, the object    (this document)
  │  model, GC, exceptions, tail calls, Unicode         ───────────────
  │  primitives                                   │
  └─────────────────────────────────────────────┘
```

The engine owns everything in L that is language semantics. The host owns
everything that is an effect or a policy. The two meet only at this API.

**Determinism (normative).** For a fixed loaded program and a fixed sequence of
host-supplied results (§7, §11), an instance's execution is reproducible: the
same sequence of positions, stack states, and values. The engine must introduce
no observable nondeterminism of its own — in particular, iteration orders visible
to Doodle code (e.g. dict traversal) and any value-identity-derived behavior must
be deterministic and independent of allocation addresses or GC timing. (See
Appendix B on the dict-order implication for L.)

**Instances (normative).** An instance owns an isolated heap and execution state.
Instances are independent and share nothing. A single instance is
**single-threaded** and is **not** thread-safe: a host must not drive one instance
from two threads at once. An instance is **reentrant**: during a synchronous
foreign call the host may call back into the same instance (§5.4, §7.6).
Different instances may run on different host threads.

---

## 3. Engine lifecycle and instances

### 3.1 Creating and destroying instances

An engine provides operations to create and destroy an instance:

```
create(config)   -> Instance
destroy(instance)
```

`config` carries, at least: resource limits (§10), the initial drive granularity
and observation mode (§7, §8), the **optional** target Unicode version (see below),
the **module resolver** hook (§6), and an opaque host-data value passed back to
foreign functions.

**Target Unicode version (resolves S-41).** The engine pins a single Unicode/UCD
version at build time — its grapheme and normalization behavior (L§4.4) — and
reports that version in its identity. The config's target-version field is
**optional**: `create` accepts a config that omits it (using the pinned version) or
names exactly the pinned version, and **fails** (a config error) on any other
requested version. The engine thus supports exactly its pinned version; a host — or
a replay driver checking a recording — asserts the expected version at create time
and gets a loud failure rather than a silent grapheme/normalization divergence
(§11; plan AD4).

`destroy` releases the instance's heap, invoking the finalizer of every live
foreign value (§4.5). After `destroy`, all handles into that instance are invalid.

### 3.2 Loading

Loading is separate from running:

```
load(instance, source, module_path) -> Module | LoadError
```

`load` lexes and parses `source` (associated with the given module path for
diagnostics and caching — the `module_path` is the entry module's **canonical id**,
the same identity imported modules get from the resolver, §6) and prepares it for execution, **without** running its
top-level code. Lexical and syntactic errors (L§3, and static errors detectable
at load such as surrogate escapes, L§3.6.3) surface here as a structured
`LoadError` carrying positions. `load` does not itself resolve `import`s; imports
are resolved lazily when top-level code runs (§6).

**Load diagnostics.** A successful load may still produce **warnings** (L§5.1
shadowing, including of prelude names). The engine keeps an instance-scoped,
monotonic **load-diagnostics record** to which it appends every diagnostic the
front end produces for every module the instance loads or attempts to load —
the entry module at `load`, and each imported module as its load executes
mid-drive (§6) — errors included. Errors keep their control-flow channels
(`LoadError` here; `module-load-error` raised in the importer, §6); the record
is the one *display* surface, read by the host with
`load_diagnostics(instance, since?)` after `load` or at any stopped state (§8),
never mid-drive. Each entry is a structured diagnostic — severity, code (the
static diagnostic slug), message, module (`canonical_id`), span, notes,
suggestion — the same shape `module-load-error` carries in its `details`. Order
is deterministic: by load order across modules, and within a module the
producer order (nondecreasing span start, production order on ties); a renderer
never re-sorts. The record is a pure function of the run's inputs (sources and
prelude exports), so it is replay-stable; it is engine-owned and host-facing —
not program data, not charged to the program's heap, not visible to Doodle code
— and lives as long as the instance (a REPL's per-chunk loads append like any
load).

A program is started by loading its entry module and then **driving** it (§7).
Running a module's top-level code is ordinary driven execution and is therefore
observable and steppable like any other code.

### 3.3 Instance state

An instance is always in one of: **ready** (loaded, not yet started, or between
top-level statements), **running** (inside a drive call), **suspended** (awaiting
a capability resolution, §7.5), **paused** (stopped at a safe point for
observation, §7.4), **completed**, **raised** (an uncaught Doodle exception
reached the outermost drive boundary, §9), or **faulted** (§9, §10). The drive
loop (§7) moves between these; each stopping outcome leaves the instance in the
same-named state (`Completed` → completed, `Raised` → raised, `Faulted` →
faulted, `Suspended` → suspended, `Paused` → paused), so `state()` alone
distinguishes a program's own error from an engine fault (§9).

**raised** is terminal, like completed and faulted: the instance is not
re-drivable (until the REPL/session facility defines re-drive semantics). In
raised, the exception value and its trace remain retained and observable
(§4.2, §8.4; the trace in §8.3's format) for post-mortem display; the call
stack itself has unwound — trap-on-raise (§8.7) is the way to observe the
pre-unwind state. A `Raised` outcome from a *nested* drive (§7.6) is the
callback's to handle and re-raises into the outer computation; only the
outermost drive's outcome moves the instance to raised.

---

## 4. The value model at the boundary

### 4.1 Values

The values that cross the boundary are exactly the language's values (L§4) —
`Int`, `Float`, `Bool`, `String`, `Bytes`, `Nil`, `List`, `Dict`, records,
callables, modules, types, protocols — plus **foreign values** (§4.5). The host
manipulates them only through **handles**.

### 4.2 Handles

A **handle** is a host-held reference to a value in an instance's heap. A live
handle keeps its value reachable (it is a GC root). The host must **release**
handles it no longer needs; unreleased handles are leaks. Handles belong to their
instance and must not be used with another instance.

```
retain(handle) -> handle          # additional reference
release(handle)
```

Reference bindings may offer scoped/automatic handles (§12); the abstract
contract is manual retain/release.

### 4.3 Constructing and reading values

The engine provides constructors and readers for each built-in kind, e.g.
(illustrative, names are binding-specific):

```
make_int(i)          make_float(x)      make_bool(b)      make_nil()
make_string(bytes)   make_bytes(bytes)  make_list()       make_dict()
make_error(kind, message, details)   # the built-in Error record (§9, L§12.1)
kind_of(h) -> Kind
as_int(h) as_float(h) as_bool(h)
list_length(h)  list_get(h, i)  list_append(h, v)
dict_get(h, k)  dict_set(h, k, v)  dict_keys(h)  dict_has(h, k)
```

**Strings (normative).** String construction and reading are the concrete home of
the String↔UTF-8 primitive the language relies on (L§4.4, L§15):

- `make_string(bytes)` validates that `bytes` is well-formed UTF-8 (raising or
  returning an error otherwise) and normalizes to NFC. The result is a `String`
  in NFC.
- `string_bytes(h)` returns the string's NFC UTF-8 `Bytes`.
- These round-trip: `string_bytes(make_string(b)) == b` when `b` is already
  valid NFC UTF-8; otherwise the result is the normalized form.

Grapheme-level and code-point-level access are not primitives here: grapheme
access is ordinary iteration (L§4.4), and code points are derived from the byte
view in the standard library (L§15). The engine does own the Unicode-table
operations (NFC, grapheme segmentation, case/property) as internal primitives; it
exposes their results through value operations and the standard library, not as
raw tables.

**Floats (normative).** `Float` values are IEEE-754 binary64, with one
restriction: the engine maintains **exactly one NaN**. `make_float(x)`
canonicalizes any NaN input to the canonical quiet NaN (bit pattern
`0x7FF8_0000_0000_0000`), every internal operation that produces a NaN
produces that canonical NaN, and `as_float` therefore only ever exposes that
pattern. NaN payload and sign bits are not representable in Doodle and never
cross the boundary. This is a determinism requirement, not a convenience:
hardware differs in the NaN bit patterns it produces for the same operation,
so un-canonicalized NaN bits would be hidden platform state — observable
through hashing, formatting, or byte views — and would break cross-host
replay (§11). Negative zero is **not** canonicalized: `-0.0` is a distinct,
fully deterministic bit pattern (equal to `0.0` under L§4.13) that
round-trips through the boundary unchanged. (Under L§4.2's finite-result
rule, the language's own arithmetic never produces a nonfinite value — it
raises instead — so the canonicalization above governs the boundary and
native/host-computed floats; host-injected ±∞ and canonical NaN are legal,
inert data.)

### 4.4 Structural inspection

The engine provides **pure, side-effect-free** structural inspection of any value:
its kind, and for compound values its structure — a record's type and field
names/values, a list's length and elements, a dict's keys and values, a string's
text and byte length, a number's value, a foreign value's type tag. Structural
inspection must not run Doodle code. It is the basis of debugger value display
(§8.4) and is distinct from the `Stringable`/`to_string` rendering hook (L§15),
which *does* run Doodle code and may have effects or fault.

### 4.5 Foreign values

A host may expose a host object to Doodle as a **foreign value**:

```
make_foreign(instance, type_tag, host_ptr, finalizer?) -> handle
foreign_tag(h) -> type_tag
foreign_ptr(h) -> host_ptr
```

A foreign value is **opaque** to Doodle: it is reference-typed (its equality is
identity, L§4.13 via `is_same`), it may be held, passed, and stored, but it has
no fields and does not support `.field` or `[]` (attempting them raises, per
L§6.2/§6.3). The host recognizes its own foreign values by `type_tag`. The
optional `finalizer` releases the underlying resource, given the value's
`host_ptr`.

**Finalizer timing (exactly once).** A finalizer runs **exactly once**: at the
garbage collection that reclaims the value — after that collection completes — or
at `destroy` (§3.1) for a value still live then, never both and never twice.
Finalizers run **host-side only** and are never observable to Doodle: a finalizer
cannot re-enter the instance, so whether — or in what order — finalizers run does
not affect any program result or the engine's determinism (§11). A finalizer
**must not fail into the instance** (it runs outside any Doodle drive) nor unwind
out of the host callback; the engine isolates a misbehaving finalizer so it cannot
prevent its peers from running or tear down the host, but the resource it was to
release may then leak. The in-engine finalizer representation is provisional
pending the C-ABI host-callback FFI (App C, S-42).

Foreign values are how host state that cannot be a plain Doodle value — a canvas,
a file handle, a rendering target, a network socket — crosses the boundary.
Doodle-side libraries typically wrap a foreign value inside a record (which *can*
carry behavior via protocols) whose operations call foreign functions on the
underlying handle.

Foreign values do not participate in protocol dispatch in this version (Appendix
B); polymorphism over host objects is expressed on the wrapping record.

---

## 5. Foreign functions and native modules

### 5.1 Foreign functions

A **foreign function** is a host callback exposed to Doodle. It is registered with
a descriptor:

- **kind** — procedure (`to`) or function (`fn`), honoring L§8.4: a foreign
  procedure yields no value (using its call in expression position is an error,
  L§6.11); a foreign function yields a value.
- **parameters** — the ordinary parameters (names, and any defaults) and at most
  one trailing block parameter (L§8.2). The engine performs call-site argument
  binding (positional/keyword, defaults, block) per L§8.3 **before** invoking the
  host, so the callback sees arguments already resolved by name and position.
- the **callback** (for synchronous foreign functions, §5.2) or a **capability
  identity** (for suspending capabilities, §5.3).

### 5.2 Synchronous foreign functions

A synchronous foreign function is invoked inline by the engine. Its callback
receives an *activation* granting access to the bound arguments (as handles), the
instance, and the host data, and must do one of:

- return a value handle (for `fn`) or return nothing (for `to`);
- raise a Doodle exception (§9), given an exception value; or
- **call back** into Doodle — invoke a callable, or invoke its block argument —
  which drives the instance reentrantly (§5.4, §7.6).

Synchronous foreign functions are for effects that do not need to yield to the
host's scheduler: pure computation, non-blocking output, and host-provided
higher-order functions (a native `each`-like primitive that invokes a block).

### 5.3 Suspending capabilities

A capability is a foreign operation that must yield control to the host before it
can produce a result — because it blocks (waiting for animation to finish, for
input, for I/O) or because the host's scheduler is an event loop that cannot be
blocked (the browser main thread). When Doodle calls a capability, the engine does
**not** invoke a callback inline; it **suspends** and returns a `Suspended`
outcome carrying a **capability request** (§7.5). The host performs the effect —
synchronously or asynchronously, on whatever thread or event loop it chooses — and
**resolves** the request, supplying the value that becomes the call's result (or
an error to raise). Execution then continues.

A capability suspends **at most once** per call: it is surfaced, resolved, and
produces its result. A native operation that must block several times is composed
from several capability calls (in Doodle, or via reentrant driving). This keeps
capabilities simple state-free requests rather than resumable host coroutines.

Whether a given primitive is synchronous (§5.2) or suspending (§5.3) is the
host's choice: a host may implement `print` synchronously if it never blocks, or
as a capability if it wants output routed through its event loop. Animated
`forward` and `read_line` are naturally capabilities.

### 5.4 Reentrancy

While handling a synchronous foreign call, the host may invoke a Doodle callable
(e.g. to call a block argument). This is a **nested drive** on the same instance
and the same stack: the engine runs the callable **to completion**, returning an
outcome to the host, which eventually returns from the foreign call. A nested drive
may additionally return **`NonLocalExit`** when the invoked block exits (a
`break`/`return`) to a target outside the host call, which the callback must relay
by returning promptly (§7.6). It **cannot** suspend: a capability reached inside a
nested drive faults `NestedSuspend` (§7.6, S-15), because the native consumer's
in-progress state lives on the host stack and cannot be frozen and resumed. Nested
drives observe the same instrumentation as top-level drives (§8). Reentrancy is
single-threaded; the host must not drive the instance from another thread
meanwhile (§10).

### 5.5 Native modules

A host exposes a capability library as a **native module**: a module value whose
members are foreign functions, constants, foreign values, or records — and
nothing else: a native module declares no dynamic `parameter` cells, protocols,
or protocol implementations. Those are language constructs, provided by a
Doodle wrapper module over the native primitives (the turtle library is the
model); a native function that depends on dynamic state receives it as an
argument from its wrapper, so `with`'s dynamic extent never crosses the
boundary and `with`-binding an imported parameter always targets a Doodle
module's cell. Native
modules are registered with the instance and are found by module resolution (§6)
ahead of source lookup. Their members honor L export semantics: a native module
declares which members are public. `import`ing a native module and accessing its
members is indistinguishable, to Doodle, from importing a module written in
Doodle.

**Provisional: pre-module intrinsic registration (retired at the module
system + prelude).** Until modules and the standard-library prelude exist, a
host registers individual **intrinsic foreign functions** (e.g. `print`)
directly with the instance, **before the instance's first `load`**.
Registering after that point, or registering a name that duplicates a prior
registration or a built-in type value, is a host-API error. Each intrinsic is
seeded into the global namespace as a **read-only** binding appended after
the program's module-level declarations, so a program's own declaration of
the same name shadows the intrinsic — exactly the relationship a module's own
declaration will have to the prelude star-import that replaces this mechanism
(L§11.4); assignment to one is a static error, as for any name that is not a
visible mutable `let`. Registration order is part of replay identity (§11).
Retirement: the host-configured prelude star-import replaces this seeding
with **no program-observable change** — the same names, read-only, shadowed
the same way — so programs and conformance fixtures written against seeded
intrinsics run unchanged across the switch.

---

## 6. Module resolution

The engine implements `import` (L§11); the host supplies the source. When top-level
or imported code executes an `import` for a module path not already loaded and not
a registered native module (§5.5), the engine **suspends with an import request** —
a capability-style request (§7.5) whose identity is the dotted module path (with
the importer's `canonical_id` for diagnostics) — and the host resolves it:

```
Suspended(ImportRequest(module_path, importer))
resolve(instance, Source(text, canonical_id))   # the module's source
resolve(instance, NotFound)                      # raises `module-not-found` in the importer
resolve(instance, Raise(h))                      # raises `h` in the importer (e.g. a failed fetch)
```

Import is a suspension rather than a synchronous hook so that it obeys the same
law as every other host interaction (§2, §5.3): a host whose source fetch is
asynchronous (a browser fetching over the network) resolves when the source
arrives, and a host that has the source in hand — a bundling host — resolves
immediately, inside the same drive loop it runs for capabilities. Nothing in the
contract requires yielding to an event loop, so the synchronous host is the
trivial case, not a different mode. The host owns the search path, virtual file
system, and bundling; the engine never touches a file system. The `canonical_id`
identifies the module for caching and diagnostics. On `Source`, the engine parses
and then **drives** the module's top-level code (observable), caches the resulting
module instance (**singleton** loading, L§11.3), and binds names per the import
form; the importer stays parked beneath the module's frame throughout. Import
requests are recorded and replayed like any capability resolution (§11), so a
recording carries every loaded module's source or identity in load order.
Because `import` is a module-level statement executed in a top-level frame, an
import request never arises inside a nested drive, so the nested-drive
suspension rule (§5.4) does not apply to it.

The engine detects **circular imports** (L§11.3) and raises. Packages (directories,
L§11.1) are the host's concern: the engine passes the dotted path through to the
resolver, which maps it to a source.

A **load failure** — the module's top-level raised, or its `Source` did not compile
(`module-load-error`, the runtime face of §3.2's `LoadError`) — marks the module
*failed* and **retains the exception it produced**; the raise propagates to the
importer at the `import` statement. A later import of the same module **re-raises the
retained value unchanged** (L§11.3) rather than re-running the load, so a recording
replays a failed load identically (§11). Because a load failure cannot be caught in
the same run — an `import` is a top-level statement, never inside a `try` — this
terminates the program; the retained value is re-raised only when an environment
feature (reload) re-imports.

Module loading runs code and can therefore suspend, pause, raise, or hit a limit
like any other driven execution; the host sees these through the ordinary drive
loop (§7).

---

## 7. The execution model — the resumable drive loop

### 7.1 The engine is resumable

The engine is a **resumable state machine** the host advances. It does not run on
a thread it owns and does not block: when it cannot or should not proceed, it
returns control to the host with an **outcome** describing why, and the host later
resumes it. This single mechanism serves running, blocking capabilities, and
debugging, and it is what lets the engine run on a non-blockable host thread (the
browser main thread) and be recorded for replay (§11).

The engine maintains its execution state — including the call stack — on the heap,
not on the host's call stack. (This is also required by proper tail calls and by
stack introspection, L§8.7, §8.2, so resumability adds little beyond machinery the
engine needs anyway.)

### 7.2 Outcomes

Driving an instance returns an **outcome**:

```
Outcome =
  | Completed(value?)                 # the driven unit finished (top-level, or a
  |                                   #   reentrant callable returned); value for fn
  | Suspended(request)                # a capability must be fulfilled by the host
  | Paused(reason)                    # stopped at a safe point for observation
  | Raised(exception, trace)          # an uncaught exception reached the boundary
  | Faulted(fault)                    # a limit, cancellation, nested-suspend, or internal fault

PauseReason = Step | Breakpoint(id) | RaiseTrap | HostPause | SliceEnd
EngineFault = LimitExceeded(kind) | Cancelled | NestedSuspend | Internal
```

**`NestedSuspend`** is the S-15 fault (§7.6): a suspending capability was reached inside
a native block-consumer's reentrant drive, which the engine forbids (terminal,
deterministic) rather than freeze-and-resume the native consumer's host-stack state.

The `value` on **Completed** is present exactly when the driven unit is a returning
`fn` — a reentrant callable return (§7.6). A **top-level module drive completes with
no value (Void)**: a module runs for effect, and statements yield no value (L§6.11).

**SliceEnd** is an ordinary `Paused`: it stops at a safe point with fully inspectable,
resumable state (§7.4), the host may observe (§8), and any drive call — bounded or
unbounded — resumes it. It reports that a **bounded-run slice** exhausted its fuel
(§7.3), a *host-scheduling* boundary distinct from `HostPause`, which is a genuine host
*request*.

### 7.3 Driving and resolving

The host advances an instance with two conceptual operations (their exact factoring
is binding-specific, §12):

```
run(instance, directive) -> Outcome              # start, or continue after a Paused
resolve(instance, resolution) -> Outcome         # continue after a Suspended
run_slice(instance, directive, fuel) -> Outcome  # run, bounded to `fuel` safe points
resolve_slice(instance, resolution, fuel) -> Outcome   # resolve, bounded
resolution = Value(handle) | Raise(handle)

directive = RunToCompletion   # stop only on Suspended / Raised / Faulted / Completed
          | Continue          # additionally stop on Breakpoint / RaiseTrap
          | Step | StepInto | StepOver | StepOut   # stop at the next matching safe point
```

- `RunToCompletion` ignores breakpoints and step boundaries (a "fast" run), stopping
  only for capabilities, uncaught errors, faults, or completion.
- `Continue` runs until the next capability, breakpoint, raise-trap, fault, or
  completion — the normal "resume the program" of a debugger.
- The `Step*` directives stop at the next safe point selected by frame depth
  (§8.5).

After a `Suspended`, the host performs the effect and calls `resolve`; execution
continues under the directive in force. After a `Paused`, the host inspects (§8)
and calls `run` with a new directive. After `Completed`, `Raised`, or `Faulted`,
the driven unit is finished.

**Bounded-run fuel (resolves App C S-40).** The `_slice` forms run the same drive under
the same directive but **bounded to `fuel` statement safe points** (§7.4); when the
fuel is spent at a safe point they return `Paused(SliceEnd)`. This is the primitive a
host pumps a jank-free browser main thread with (plan AD6): `run_slice(instance,
RunToCompletion, n)` is the fast path — a bounded fast run with no debug stops.

- **Fuel is orthogonal to the directive.** *Any* directive may be sliced; fuel is a
  separate operand, not part of the `directive` enum.
- **Per drive call, never banked.** Each `_slice` call brings its own fuel; a plain
  `run`/`resolve` after a `SliceEnd` runs unbounded. `fuel = 0` returns `Paused(SliceEnd)`
  immediately, running nothing (a legal observe-without-progress poll). This contrasts
  with the **directive**, which is *sticky* across a `Suspended`/`Paused`; fuel is not.
- **Related rails, different roles.** Slice fuel counts statement safe points, while the
  step budget (§10.2) counts work units, of which a statement safe point is one: the rails
  share the statement unit, but only the budget carries operation charges — an atomic
  operation cannot yield mid-way, so a slice charge would not aid responsiveness; the
  budget's pre-charge is what bounds the longest single operation. The step budget is a
  **terminal** limit (`Faulted(LimitExceeded)`) and the slice fuel a **resumable
  scheduling** bound (`Paused(SliceEnd)`). When both would trip at the same safe point, the
  **fault wins**.
- **A terminal transition wins the race** (as for cancellation, §10.1): a **control
  signal observed at a terminal transition defers to it** — if the fuel is spent (or a
  cancellation observed) at the very transition that completes the program or lands it
  `Raised`/`Faulted`, the run's own terminal outcome stands; a `SliceEnd` would falsely
  claim work remains.
- **Program-invisible, outside replay identity (§11).** A sliced run and an unbounded run
  produce **identical program-observable execution** — the language has no clock, so
  `SliceEnd` is undetectable from within Doodle. Slice boundaries are host scheduling,
  not recorded input.

### 7.4 Safe points

A **safe point** is a location where the engine may stop with fully inspectable and
resumable state. The engine must place safe points at least:

- between statements (L§7), and
- at call entry and at return.

These **statement-level** safe points are where the engine evaluates stop
conditions — breakpoints, the current step target, a host pause request — **and
where resource accounting happens**: resource limits (§10), cancellation
observation (§10.1), the step budget, and slice fuel are evaluated here and only
here, in every observation mode. Between safe points the engine's internal state
need not be inspectable. The default observation granularity is per-statement
(§8.1).

**Fine observation mode** (§8.8) adds **observation-only** safe points: one at the
**completion of every non-leaf subexpression** — each operator application, each
call (besides its entry and return), each field access and index step, each
`if`-expression branch result, each interpolation piece. Leaves — literals and
name reads — are not safe points: no engine state lies between them to observe,
and their values are visible through inspection or as operands at the enclosing
completion. At a fine safe point the host observes the completed subexpression's
position (§8.1) and **the value it just produced** (§8.4) — the primitive for
watching an expression evaluate. Fine safe points are only places a drive may
return `Paused(Step)` or `Paused(HostPause)`; no accounting, GC, or limit check
runs there, so a fault or budget exhaustion lands at the same instant in either
mode (§7.7). The fine safe-point set is defined by syntactic form, above, and is
part of the engine's replay identity: the same program and directives stop at
the same positions in every run. Its cost — one check per subexpression
completion — is paid only while fine mode is on; coarse mode pays nothing.

### 7.5 Capability requests

A `Suspended` outcome carries a **capability request**: the capability's identity
(which registered capability, §5.3) and its bound arguments (as handles). The host
fulfills it and resolves with `Value(h)` — which becomes the capability call's
result — or `Raise(h)` — which raises `h` at the call site. A `to` capability is
resolved with a `Value` of `nil` or an equivalent no-value resolution. The
capability identity must be stable across runs so that resolutions can be recorded
and replayed (§11). An `import` (§6) suspends with a request of the same kind — its
identity the module path — resolved with `Source`/`NotFound`/`Raise` rather than
`Value`.

### 7.6 Reentrant drives

A synchronous foreign function (§5.2) may drive the instance reentrantly by invoking
a Doodle callable — or the **block argument** it received (§5.1, L§8.5). The nested
drive runs on the instance's single heap stack and shares its instrumentation. It
runs the invoked callable **to completion** and returns its own outcome to the
caller — normal completion, a raise, a **`NonLocalExit`** (below), or a fault. It
does **not** pause or suspend outward: a suspending capability reached inside it is
the forbidden S-15 case (below).

**Non-local exits across a native consumer (resolves App C S-46).** A `do … end`
block invoked by a native block-consuming function may execute an exit whose target
lies *outside* the block. There are three cases, mirroring how the three outcomes of
an ordinary block invocation surface to a Doodle consumer:

- A **`continue`** (and normal fall-off) targets the block itself: the invocation
  simply **completes** — `Completed(value?)` to the callback — so the callback runs
  its next iteration. It is never a `NonLocalExit`.
- A **`break`** targets the call that *received* the block (L§7.10), and a
  **`return`** targets the enclosing Doodle function — both on the far side of the
  host call. The nested drive cannot complete these directly, so it returns a
  distinguished **`NonLocalExit(kind)`** outcome (kind `break`/`return`, carrying a
  value for a valued exit) — the transport-level analogue of a `Raised` crossing a
  native frame (§9).

On receiving a `NonLocalExit`, the foreign callback **must return promptly with no
result** (host-side cleanup only — it may **not** re-invoke the block or start a
nested drive; Doodle-level cleanup runs in the engine's own unwind, not the host's).
The engine then resumes the parked exit at the foreign call's apply site: a `break`
targeting *that* call completes it with the exit's value; a `return` (or a `break`
aimed at a construct enclosing this one) keeps unwinding past the call toward its
true target, so several nested native consumers each see the `NonLocalExit` in turn,
innermost outward, the unwind resuming at each apply site. A host that returns a
value, raises, or drives again after `NonLocalExit` violates the contract and
**faults** the instance — the exit's target integrity cannot otherwise be preserved.
(A related host-contract case — a callback **abandoning** a nested drive it has not
driven to completion — is **S-16**. Under forbid-and-fault a block-invocation nested
drive always completes, raises, exits (`NonLocalExit`), or faults `NestedSuspend`, so the
specific "nested drive still `Suspended`/`Paused`" phrasing cannot arise via block
invocation today; S-16 becomes live only if the M7 suspend-the-outer-drive extension lets
a nested drive suspend.) `NonLocalExit`
and its resumption are engine-derived from the running program, not host input, so
they add nothing to the recordable boundary (§11).

**Suspending inside a nested drive (resolves App C S-15).** A **suspending capability**
(§5.3) reached while a nested drive is running — a block invoked by a native
block-consumer calls one — cannot suspend the way a top-level capability does: the
native consumer's own progress (a loop index, say) lives on the **host's call stack**,
which the engine cannot freeze and later resume. The engine therefore **forbids** the
nested suspend: it is a **non-resumable `Faulted(NestedSuspend)`** (§7.2), terminal and
deterministic — a well-defined engine limitation reached by legitimate Doodle code,
distinct from an `Internal` invariant violation. (A capability reached in an ordinary,
non-nested drive suspends normally — **including inside a *Doodle* block-consumer**,
whose block runs on the engine's own heap stack; only a **native** consumer's reentrant
drive is affected.) The alternative — *suspending the outer drive* by propagating the
request across the native boundary and making native consumers **resumable** (saving and
restoring their progress across a suspend) — is the same explicit save/resume protocol a
C foreign function needs (it cannot freeze its C stack either), so it is **deferred to the
C-ABI design (M7)** rather than pre-committed here. A host that wants a block-consumer
whose block may suspend writes it in **Doodle**, not as a native function.

### 7.7 Determinism of driving

Given a loaded program and a fixed sequence of `resolve` resolutions, `run`/`resolve`
produce an identical execution regardless of directives: directives affect *where the
host regains control*, never *what the program computes*. Stepping never changes
results.

---

## 8. Observation and instrumentation

All of the following are available on a **paused** or **suspended** instance, at a
safe point, without running Doodle code (except where noted). They are **pull**
operations: the host inspects a stopped instance rather than receiving callbacks
mid-evaluation, which avoids reentrancy hazards. Event-like behavior is obtained by
driving in a `Step*` directive. The load-diagnostics record (§3.2) is additionally
readable on a `ready` instance, immediately after `load`.

### 8.1 Current position

The engine exposes the **current source position**: the module's `canonical_id` and
the line/column span (columns in code points, per L§3.1) of the statement (or, in
fine mode, the subexpression) about to execute or just executed. The engine exposes positions, not source text;
the host holds the source it loaded and renders it (consistent with L§13, which
exposes provenance but not source/AST).

### 8.2 The call stack

The engine exposes the current **call stack** as an ordered list of **frames**,
innermost first. Each frame exposes:

- the **callable** (a handle; its reflection data — name, procedure/function kind,
  source location, docstring — is available per L§13);
- the **call-site position** (where this frame was entered);
- the frame's **local bindings** — parameters and `let`/`const` names in scope,
  as name→value handles;
- the **dynamic-parameter bindings** established by `with` within this frame (L§5.5),
  as name→value handles.

### 8.3 Tail-call provisions

Because proper tail calls reuse frames (L§8.7), the live call stack omits
tail-elided callers. To keep traces and recursion visualization meaningful, the
engine additionally exposes:

- a **bounded tail-call history**: a ring buffer of recently elided frames, each
  marked `tail_elided` and carrying its callable and call-site; the bound is set in
  `config` (§3.1);
- per live frame, a **tail-iteration count**: how many times that frame has been
  reused by a tail call, so a visualizer can show "`polygon` called itself 47 times"
  without an unbounded stack.

The live frames and the tail-elided history are distinguished in the API; a host
must not present elided frames as if they were live activations, but should use the
history and counters to convey that tail recursion occurred. This realizes the
"bounded history of elided tail frames" that L§8.7 requires.

### 8.4 Value inspection

Given any value handle, the host obtains its structure via the pure structural
inspection of §4.4 — kind, and for compounds the fields/elements/keys — **without**
running Doodle code. A debugger renders program state from this. A host may
additionally obtain the program's own rendering of a value by driving `to_string`
on it (L§15), but that is an explicit, effectful action (it runs Doodle code and may
fault), not part of inspection.

### 8.5 Stepping

Stepping is expressed through the drive directives (§7.3), defined against safe
points and the frame depth at the moment the directive is issued:

- **StepInto** — stop at the next safe point in any frame (including a newly entered
  callee).
- **StepOver** — stop at the next safe point in the current frame or a caller (do not
  stop inside callees).
- **StepOut** — stop at the next safe point in a caller (run the current frame to its
  return).
- **Step** — a synonym for `StepInto` unless a host distinguishes them.

Tail calls interact with stepping: a `StepOver` across a tail call observes that the
current frame is replaced rather than returned-into; the engine treats the reused
frame as "the same or shallower depth" so that `StepOver`/`StepOut` behave as a user
expects for a tail-recursive loop (they do not run away).

### 8.6 Breakpoints

The host may set and clear **breakpoints** at a source position (module `canonical_id`
+ line):

```
set_breakpoint(instance, canonical_id, line) -> id
clear_breakpoint(instance, id)
breakpoints(instance) -> [{id, canonical_id, line, resolved}]
```

The module is addressed by its **canonical id** — the host-owned identity the
resolver mints (§6) and `load` records for the entry module (§3.2); engine-internal
module indices are not part of the boundary. A breakpoint whose canonical id names a
module not yet loaded — or a file never imported at all — is **pending**, not an
error: the set-then-run flow (mark the gutter, press Run) must work for modules that
load mid-drive. Breakpoints **re-resolve at every load of their canonical id**:
resolution snaps forward to the first safe point at or after the line (first on the
line wins); a line with no safe point at or after it leaves the breakpoint pending
and unhittable. Re-resolution is also the canonical-id-reuse rule — a reloaded module
(a REPL session) keeps its breakpoints, re-snapped against the new source. The
listing reports each breakpoint as resolved or pending so a host can gray unhittable
marks.

Under a `Continue` or `Step*` directive the engine stops with `Paused(Breakpoint(id))`
at the first safe point at or after a breakpointed position. Under `RunToCompletion`
breakpoints are ignored. Breakpoints are host directives, like stepping: outside
replay identity, covered by drive-directive determinism (§7.7). Conditional breakpoints (a Doodle predicate evaluated at the
safe point) are a noted extension (Appendix B), since evaluating a condition runs
Doodle code.

### 8.7 Trap on raise

The host may enable **raise-trapping**. When enabled, and under a `Continue` or
`Step*` directive, the engine stops with `Paused(RaiseTrap)` at the point an exception
is raised (L§12.1), *before* the stack unwinds, so the debugger can inspect the raising
frame with the stack intact. Resuming continues the unwind (the exception propagates
normally). Raise-trapping is independent of whether the exception is later caught.
`RunToCompletion` ignores raise-trapping, as it ignores breakpoints (§8.6) — it stops
only for the outcomes §7.3 lists; enabling a trap says *which* stops interest the host,
the directive says whether debug stops fire at all.

### 8.8 Host-requested pause and observation mode

The host may request a pause; the engine stops with `Paused(HostPause)` at the next
safe point. An instance's **observation mode** has one axis: safe-point granularity —
per-statement or per-subexpression (§7.4) — set in `config` and adjustable, letting a
host trade fidelity for speed: run `RunToCompletion` with coarse safe points when
nobody is watching, switch to fine stepping when the user opens the debugger.
Switching mode changes only where stepping may stop (§7.4); it never changes what the
program computes or when a limit trips. There is no eager-capture axis: inspection is
pull (§8) — the host reads live frame state on demand when stopped, which is as lazy
as possible and costs nothing while running — and reading state that no longer exists
(a popped frame's locals, step-back history) is the deterministic-replay track's job
(§11), not an observation mode.

### 8.9 Live edit (optional)

Writing a value into a frame binding, or replacing a definition mid-run
(Smalltalk-style live editing), is an **optional** capability. A conforming engine
may expose binding-write and hot-redefinition; a minimal engine exposes read-only
inspection. This is deferred (Appendix B).

---

## 9. Errors and protected execution

- **Doodle exceptions** (L§12) are values. A foreign function raises one by supplying
  an exception value (§5.2); the engine unwinds Doodle frames normally, restoring
  dynamic-parameter bindings and running block-protocol cleanup as it goes (L§5.5,
  L§12.2).
- **Engine-raised errors are `Error` records** (L§12.1): a built-in value record
  `Error(kind, message, details)` — a stable kebab-case `kind` slug (the catalog is
  part of the engine's stable surface, implementation plan App C S-58), a readable
  `message`, and a `details` dict of kind-specific structured data hosts render and
  localize from. `make_error(kind, message, details)` (§4.3) builds the same value,
  so foreign functions and native intrinsics raise the same shape the engine does;
  a foreign function may still raise any value. The value carries *what* went
  wrong; *who* raised it — an engine primitive or a program's `raise` — is in the
  trace, so `Error` values are deliberately forgeable by Doodle code.
- An uncaught exception that reaches a drive boundary yields `Raised(exception, trace)`.
  The **trace** is captured at `raise` (L§12.1) and includes the live frames and the
  bounded tail-elided history (§8.3) as positions and callables. When the boundary is
  the outermost drive, the instance enters the terminal **raised** state (§3.3),
  retaining the exception and trace for post-mortem inspection.
- **Driving is inherently protected.** Every `run`/`resolve` returns an outcome; a
  Doodle error never propagates as a host-language exception or crash. (This subsumes
  the role of a `pcall`-style protected-call primitive.)
- **Engine faults** — resource-limit exhaustion, cancellation, a nested-drive suspend
  (`NestedSuspend`, §7.6), or an internal invariant violation — are `Faulted`, distinct
  from Doodle exceptions and **not** catchable by Doodle code.
- A foreign function that fails in a way the program should handle raises a Doodle
  exception; a host-level failure it cannot express becomes an engine fault surfaced
  to the driving host.

---

## 10. Control, limits, and concurrency

### 10.1 Cancellation

The host may request **cancellation** of an instance — the "stop button". Cancellation
is a request about *future* work, not a verdict on *past* work; the guarantee it makes
is that **once a cancellation request is observed, no further program work runs.**

The engine polls the request at each safe point (§7.4). Observed at a safe point with
program work still ahead — the running or suspended case — the engine unwinds the stack,
restoring dynamic-parameter bindings and running block-protocol cleanup as for an
exception, and returns `Faulted(Cancelled)`: a terminal, non-resumable outcome (§3.3)
that Doodle code **cannot** catch.

**Cancellation takes effect only at a safe point where program work remains; otherwise
the run's own terminal outcome stands** — the general rule that a control signal observed
at a terminal transition defers to it (§7.3, shared with bounded-run `SliceEnd`). A
request first observed at the instant a program completes yields `Completed`, and one
first observed as an uncaught exception reaches the boundary yields `Raised` — not
`Faulted(Cancelled)`. In each of these the
program has already run to its true end, and reporting `Faulted(Cancelled)` would
misreport reality — claiming the engine stopped work that in fact all happened, so a
host could wrongly conclude some effect did not occur. (This is the universal
convention: cancelling a task after it has finished is a no-op.) Requesting cancellation
of an already-**terminal** instance is likewise an idempotent no-op.

**Reaping a cancelled *suspended* instance (S-23).** A suspended instance advances only
through `resolve` (§7.5), so a cancellation requested while it is suspended is reaped by
resuming it — and once resumed with program work still ahead, the engine faults
`Cancelled`, the pending capability request **discarded**, whether the host resolves with a
value or a raise. A **value** resolution resumes the parked call and drives to the next
safe point, which observes the cancellation (a resumption that *completes* the program
first instead stands as `Completed` — a cancel racing completion loses, above). A **raise**
resolution — the host rejecting the call — does **not** surface as `Raised` while a
cancellation is pending: the rejection is discarded and the stack torn down to
`Faulted(Cancelled)` without running the parked call's continuation, since a pending cancel
wins over a host raise. Once terminal, the instance is not re-drivable (§3.3, §7.3): a
*later* `resolve` or `run` is the ordinary re-drive-of-terminal host-contract violation (a
non-resumable `Faulted`), **not** a second cancellation — so the "stop button" resolves the
suspended instance once to reap the `Faulted(Cancelled)` and then does nothing more with
it. (This is the browser turtle demo's stop-mid-animated-`forward` path, M3.6.)

The **instant** at which a request is first observed is host timing, and so lies outside
replay identity (§11): after requesting cancellation a host must accept **either**
`Faulted(Cancelled)` **or** the run's own terminal outcome (`Completed`, `Raised`, or a
resource `Faulted`). A run for which cancellation is never requested is never affected.

### 10.2 Limits

An instance is configured (§3.1) with limits appropriate to an untrusted or
kid-authored program:

- a **step budget** — since the engine owns no clock, wall-clock timeouts are enforced by
  the host via the step budget or by cancelling. The budget counts **work units**: an
  ordinary statement safe point costs one unit, and an operation that can produce a result
  much larger than its operands — bignum `*`/`**`, string or list repetition — is charged,
  before it runs, a deterministic estimate of its result size in units, so no single
  operation can perform unbounded work under a bounded budget (every other operation's cost
  is bounded by the heap limit on its operands). The estimate is a pure function of operand
  values and is part of the engine's replay identity (§11): the same build faults at the
  same step;
- a **heap limit** (bytes/objects); an operation that would produce a value exceeding it —
  the same result-growing operations — faults before allocating, from the same deterministic
  size estimate, rather than attempting an allocation that could exhaust memory. When one
  operation would exceed both rails, the heap fault is reported (the result could not exist
  regardless of the work);
- a **stack-depth limit** for non-tail recursion (proper tail calls do not count, L§8.7,
  so a tail-recursive loop never trips it, but unbounded non-tail recursion does);
- the **tail-history bound** (§8.3).

Exceeding a limit yields `Faulted(LimitExceeded(kind))`. Limits are essential for an
environment running programs that legitimately contain infinite loops.

### 10.3 Concurrency and reentrancy

An instance is single-threaded and not thread-safe: a host must serialize all access
to one instance (drives, handle operations, registration) to a single thread at a time.
Reentrant driving within that thread is supported (§5.4, §7.6). Instances are fully
independent and may run concurrently on different host threads. Handles belong to their
instance and are not shareable across instances. Doodle has no language-level
concurrency (L§14); any parallelism is between independent instances, coordinated by
the host.

---

## 11. Determinism and the recordable boundary

The engine is deterministic (§2, §7.7): a loaded program plus the sequence of `resolve`
resolutions supplied for its `Suspended` outcomes fully determines its execution.
Everything nondeterministic — input, randomness, time, external events — enters only as
a resolution to a capability request. (This is why the language core has no clock or
RNG, L§14: those are capabilities, so they cross the recordable boundary.)

**Recording.** A host records, for a run: the loaded program (or its identity), and, in
order, each capability request's identity and the resolution supplied — import
requests included (§6), so every loaded module's source or identity is in the record.

**Replay.** To replay, the host creates a fresh instance, loads the same program, drives
it, and — instead of performing effects — supplies the recorded resolutions in order for
each `Suspended`. The reconstruction is identical: same positions, same stack, same
values. This yields deterministic forward reconstruction to any step (fast-forward),
and shareable replay artifacts.

**Requirements on the engine.** For replay to hold, the engine must expose stable
capability-request identities (§7.5) and must have no hidden nondeterminism (§2):
value identity, GC, and hashing must not leak observable order into Doodle; in
particular **dict iteration order must be deterministic** (see Appendix B — this is a
constraint that feeds back into L§4.8). Likewise **NaN bit patterns must be
canonicalized** (§4.3): hardware produces differing NaN payloads for the same
operation, so an un-canonicalized NaN would be hidden platform state — observable
through hashing, formatting, or byte views — and recordings would not replay across
hosts.

**Floating-point results must be host-independent (S-57).** The basic IEEE-754 operations
(`+ − × ÷`, comparison, `sqrt`) are correctly rounded and so already identical across
conforming hosts, but transcendental and other library math functions (`sin`, `cos`, …)
are **not** pinned by IEEE-754 — a platform `libm` may differ in the last bit(s), and a
hardware fused-multiply-add may round differently from a separate multiply-then-add.
The engine therefore computes every Doodle-observable floating-point function with its
**own bundled implementation, identical on every supported target** (native and
WebAssembly), and does not call the platform math library or rely on target-specific FMA
— otherwise the same program would compute different values on different hosts and
recordings would not replay across them (exactly as an un-canonicalized NaN would leak).
This assumes each supported target evaluates `f64` in **strict IEEE-754 double precision**
(`FLT_EVAL_METHOD == 0`) — true of the aarch64, x86-64+SSE2, and wasm32 targets the
engine ships; an excess-precision target (e.g. x87 without SSE2, where a soft-float
`sin`/`cos` may still round through 80-bit registers) is **out of scope** until pinned,
and adding one requires re-establishing this bit-identity. (Provisional M3: the first
such functions, `sin`/`cos`, are engine natives pending the standard library; the `**`
float path uses the same bundled `pow`. L§14/§15.)

**Deferred.** Efficient *reverse* stepping needs periodic heap **snapshots** plus replay
between them; the snapshot format, the replay/serialization API, and shareable-artifact
encoding are deferred to a later revision (Appendix B). This section fixes only the
determinism property and the boundary that make them possible.

---

## 12. Reference bindings (informative, with normative constraints)

The API is specified abstractly (the operations and outcomes above). A binding realizes
it for a concrete host language and **must** preserve the semantic contract — the value
model, the drive loop and outcomes, the observation surface, and determinism — even where
it chooses idiomatic surface (memory-management style, synchronous vs. asynchronous
function shapes).

- **C ABI (the lingua franca).** Opaque `DoodleInstance*` and `DoodleHandle`; foreign
  functions as function pointers with a descriptor; an outcome tagged union; manual
  `retain`/`release`; the module resolver and host data as callbacks/pointers in the
  config. The concrete header is a separate artifact.
- **WASM/JS.** The engine compiled to WebAssembly with a JavaScript wrapper. Handles are
  JS-side references; foreign functions are JS callbacks; **the drive loop is surfaced as
  async (Promises)** so that a `Suspended` capability integrates with the browser event
  loop and the engine runs on the main thread without blocking it. This binding is the
  primary reason the engine is resumable (§7.1).

A binding may add convenience (scoped handles, async/await sugar, structured error types)
but must not add capabilities that bypass the recordable boundary (§11) or the
observation guarantees (§8).

---

## 13. Relationship to the standard library

The standard library (a separate specification) is written mostly in Doodle but bottoms
out in **platform primitives** — foreign functions a *platform host* must provide. These
include the effectful primitives the library builds on (output such as `print`, input
such as `read_line`, time, randomness) and the representation/Unicode primitives the
language references (L§15) — the String↔UTF-8 byte view of §4.3, plus the NFC and
grapheme operations the engine owns internally.

This document defines the **mechanism**: platform primitives are ordinary foreign
functions (§5) grouped into native modules (§5.5), and the required *set* is specified by
the standard-library specification. A minimal embedder (a game engine) need provide only
the primitives its programs use; standard-library modules whose primitives are absent
simply fail to load when imported. A full **platform host** (the reference environment)
provides the complete set plus the interactive facilities of §7–§11.

---

## Appendix A. Glossary

- **Engine** — an implementation of this API; the pure core interpreter.
- **Instance / runtime** — one live engine with an isolated heap and execution state.
- **Host** — the embedding program. **Environment** — an interactive host.
- **Capability** — an effect performed by the host on the program's behalf.
- **Foreign function** — a host callable exposed to Doodle (synchronous, §5.2, or a
  suspending capability, §5.3).
- **Foreign value** — a host object exposed to Doodle as an opaque, reference-typed value
  (§4.5).
- **Handle** — a host-held, GC-rooting reference to a Doodle value (§4.2).
- **Drive** — advance execution (`run`/`resolve`, §7.3). **Outcome** — why a drive
  returned (§7.2).
- **Safe point** — a location where the engine may stop, be inspected, and be resumed
  (§7.4).
- **Suspension / capability request** — the engine yielding to the host to have a
  capability fulfilled (§7.5).
- **Frame** — an activation on the call stack (§8.2). **Tail-elided** — a frame reused by
  a proper tail call and therefore not live (§8.3).

---

## Appendix B. Decisions and open issues

### B.1 Decisions made here

- **Resumable, step-driven engine (§7).** Chosen over a blocking engine on a
  host-controlled thread. Rationale: proper tail calls and call-stack introspection
  already require an explicit heap-allocated, walkable execution stack, so resumability
  is a small addition; it makes the engine run on non-blockable host threads (the browser
  main thread), makes stepping/pausing the native interface rather than a bolted-on debug
  path, and makes deterministic replay nearly free. (Approved in discussion.)
- **Abstract contract + reference bindings (§1.1, §12).** The API is a semantic contract;
  a C ABI is the lingua franca and a WASM/JS async binding is a first-class second
  reference. Not tied to one implementation language.
- **Per-statement observation default; per-subexpression optional (§7.4, §8.1, §8.8).**
- **Always-on observability, with a fast drive mode (§8.8).** Observability is intrinsic,
  not a separate build; a host may coarsen safe points when not observing.
- **Two flavors of foreign function (§5.2, §5.3).** Synchronous callbacks for
  non-blocking work; suspending capabilities for anything that must yield to the host.
  A capability suspends at most once per call.
- **Pull-based inspection (§8).** The host inspects stopped instances; there are no
  mid-evaluation callback events (which would create reentrancy hazards).
- **Cancellation is uncatchable and unwinds cleanup (§10.1).**
- **Determinism fixed; replay/snapshot API deferred (§11).** The determinism property and
  the recordable boundary are normative now; the snapshot format, reverse-stepping, and
  serialization API are deferred.
- **Foreign values are opaque and do not dispatch protocols (§4.5).** Polymorphism over
  host objects is expressed on a wrapping record.
- **Top-level completion is Void (§7.2).** A top-level module drive's `Completed`
  carries no value; the `value` is present only for a returning `fn`. A module runs
  for effect (L§6.11). Resolves the M0.3 provisional (which returned the last
  expression's value so the M0 acceptance could observe it); landed with the M2a.2
  machine skeleton that replaced that placeholder.
- **One canonical NaN; `-0.0` preserved (§4.3, §11).** `make_float` and every
  NaN-producing operation canonicalize to the single quiet-NaN bit pattern
  `0x7FF8_0000_0000_0000`; NaN payload/sign never crosses the boundary. Required
  for cross-host replay (hardware NaN bit patterns differ) and the ground for
  L§4.13's single, self-equal NaN value (the engine side of implementation-plan
  Appendix C S-28). Negative zero is a distinct, deterministic bit pattern,
  preserved through the boundary, equal to `0.0` under L§4.13.
- **Provisional pre-module intrinsic registration (§5.5).** Until modules and
  the prelude exist, hosts register intrinsic foreign functions (e.g. `print`)
  before the instance's first `load`; they seed as read-only global bindings
  behind the program's own declarations (shadowable, like the prelude
  star-import that retires this mechanism with no program-observable change).
  Late or duplicate registration is a host-API error; registration order is
  replay-identity input (§11). Resolves implementation-plan Appendix C S-43.
- **An uncaught raise leaves the instance `raised`, not `faulted` (§3.3, §9).** A
  distinct terminal state mirroring the `Raised` outcome, so `state()` alone
  preserves §9's raise-vs-fault distinction (post-mortem hosts and IDE panels
  branch on it); the exception and trace stay observable; the module-level
  `Failed`-retaining-exception state is the same pattern. Resolves the M2a.3a
  provisional (which recorded `Faulted`); cancellation stays
  `Faulted(Cancelled)` (§10.1); a nested drive's `Raised` changes no state
  (outermost only).
- **Non-local exits cross a native block-consuming callee via `NonLocalExit`
  (§7.6, §5.4).** A `break`/`return` whose target lies outside a block invoked by a
  native function is **supported** (not disallowed): the nested drive returns a
  `NonLocalExit(kind)` outcome, the callback returns promptly with no result, and
  the engine resumes the parked exit at the foreign call's apply site (a `break`
  targeting that call completes it; a `return`/outer break keeps unwinding). A
  `continue` is not a `NonLocalExit` — it completes the block invocation normally. A
  host that returns a value/raises/re-drives after `NonLocalExit` faults. Chosen
  over disallowing such exits so a native `each`/`repeat` behaves like a
  Doodle-defined block-consumer (no-magic-boundaries, plan §1). Resolves
  implementation-plan Appendix C S-46 (user-ratified 2026-08-02).
- **Engine errors are one built-in `Error` record (§9, §4.3).** `Error(kind,
  message, details)` — engine-level type with a prelude-level name, forgeable by
  design (provenance is in the trace); `make_error` gives hosts the same shape.
  Resolves plan-m4 D-M4-2 / implementation-plan Appendix C S-58.
- **Import is a suspension, not a synchronous hook (§6, §7.5).** `import`
  suspends with an `ImportRequest`; the host resolves with `Source`/`NotFound`/
  `Raise`. Chosen over a synchronous resolver so import obeys the same law as
  every host interaction: a bundling host resolves immediately in its drive
  loop (the trivial case), a browser fetching over the network resolves when the
  source arrives, and import resolutions enter the replay record. Resolves
  plan-m5 D-M5-2 / implementation-plan Appendix C S-60.
- **Native modules carry no `parameter` cells, protocols, or implementations
  (§5.5).** Language constructs live in Doodle wrapper modules over native
  primitives; a native function receives dynamic state as an argument. Keeps
  one implementation of `with` and keeps binding machinery off the boundary.
  Resolves plan-m5 D-M5-3 / implementation-plan Appendix C S-44.
- **Fine observation mode = non-leaf subexpression completions, observation-only
  (§7.4, §8.8).** Defined by syntactic form (so hosts and replay can depend on
  it) and realized at the machine's existing continuation boundaries; leaves
  are not safe points; accounting, GC, limits, cancellation, budget, and fuel
  stay at statement safe points in every mode. Chosen over a safe point at
  every node (leaves add nothing observable) and over a config knob with no
  fine mode (an API that lies). Resolves plan-m6 D-M6-1 / implementation-plan
  Appendix C S-62.
- **One load-diagnostics record per instance (§3.2, §8).** Warnings on a
  successful load (prelude shadowing, L§5.1) and every imported module's
  load-time diagnostics accumulate in an instance-scoped, monotonic,
  deterministically ordered record with a pinned structured schema, read by
  pull like every observation; errors keep `LoadError`/`module-load-error` as
  control flow. Chosen over threading prelude names into the resolver (splits
  the mechanism) and over a main-module-only channel (a known gap). Resolves
  plan-m5 D-M5-6's channel question and the M1.1 discovered deltas (warnings
  channel, diagnostic schema, diagnostic ordering); implementation-plan
  Appendix C S-63.
- **Breakpoints address (canonical_id, line); pending + re-resolve-on-load
  (§8.6, §3.2).** The host-owned canonical id is the boundary identity (engine
  module indices stay internal); an unloaded target is pending, not an error;
  every load of a canonical re-resolves its breakpoints (snap-forward,
  first-on-line) — which is also the reuse/reload rule. Resolves
  implementation-plan Appendix C S-21.

### B.2 Open issues, including cross-spec implications

- **Dict iteration order (cross-spec, L§4.8).** Deterministic dict iteration is required
  for replay (§11); the language spec pins dict iteration order to **insertion order**
  (L§4.8), which satisfies it. Recorded here because the dependency runs from this
  document into the language spec.
- **Conditional breakpoints and live edit (§8.6, §8.9).** Both involve running Doodle code
  during observation (evaluating a condition; writing bindings/hot-redefining). Deferred;
  the mechanism (drive a predicate; write a frame binding) is sketched but not specified.
- **Snapshot format and reverse time-travel (§11).** Deferred.
- **Capability-request identity scheme (§7.5, §11).** The exact, stable, portable naming
  of capability requests (so recordings replay across engine builds) is deferred.
- **Concrete C header and JS surface (§12).** Separate artifacts.
- **Multi-suspension native functions (§5.3).** Handled by composition in this version;
  revisit if a real capability cannot be decomposed.
- **Handle lifetime idioms (§4.2).** Manual retain/release is the abstract contract;
  scoped/automatic handles are left to bindings.
- **Foreign values in protocols (§4.5).** Whether a host object may directly implement a
  Doodle protocol (rather than via a wrapping record) is deferred.
- **The platform-primitive set (§13).** Belongs to the standard-library specification;
  named here only as a mechanism.
