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
and observation mode (§7, §8), the target Unicode version (L§4.4, whose grapheme
and normalization behavior the engine pins), the **module resolver** hook (§6),
and an opaque host-data value passed back to foreign functions.

`destroy` releases the instance's heap, invoking the finalizer of every live
foreign value (§4.5). After `destroy`, all handles into that instance are invalid.

### 3.2 Loading

Loading is separate from running:

```
load(instance, source, module_path) -> Module | LoadError
```

`load` lexes and parses `source` (associated with the given module path for
diagnostics and caching) and prepares it for execution, **without** running its
top-level code. Lexical and syntactic errors (L§3, and static errors detectable
at load such as surrogate escapes, L§3.6.3) surface here as a structured
`LoadError` carrying positions. `load` does not itself resolve `import`s; imports
are resolved lazily when top-level code runs (§6).

A program is started by loading its entry module and then **driving** it (§7).
Running a module's top-level code is ordinary driven execution and is therefore
observable and steppable like any other code.

### 3.3 Instance state

An instance is always in one of: **ready** (loaded, not yet started, or between
top-level statements), **running** (inside a drive call), **suspended** (awaiting
a capability resolution, §7.5), **paused** (stopped at a safe point for
observation, §7.4), **completed**, or **faulted** (§9, §10). The drive loop (§7)
moves between these.

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
optional `finalizer` is called when the value is garbage-collected or when the
instance is destroyed, letting the host release the underlying resource.

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
and the same stack: the engine runs the callable to completion (or to a suspend
or a stop), returning an outcome to the host, which eventually returns from the
foreign call. Nested drives observe the same instrumentation as top-level drives
(§8). Reentrancy is single-threaded; the host must not drive the instance from
another thread meanwhile (§10).

### 5.5 Native modules

A host exposes a capability library as a **native module**: a module value whose
members are foreign functions, constants, foreign values, or records. Native
modules are registered with the instance and are found by module resolution (§6)
ahead of source lookup. Their members honor L export semantics: a native module
declares which members are public. `import`ing a native module and accessing its
members is indistinguishable, to Doodle, from importing a module written in
Doodle.

---

## 6. Module resolution

The engine implements `import` (L§11); the host supplies the source. When top-level
or imported code executes an `import` for a module path not already loaded and not
a registered native module (§5.5), the engine calls the host's **module resolver**
hook:

```
resolve(module_path) -> Source(text, canonical_id) | NotFound
```

The host owns the search path, virtual file system, and bundling; the engine never
touches a file system. The `canonical_id` identifies the module for caching and
diagnostics. On `Source`, the engine parses and then **drives** the module's
top-level code (observable), caches the resulting module instance (**singleton**
loading, L§11.3), and binds names per the import form. On `NotFound`, the engine
raises an import error in the importing program.

The engine detects **circular imports** (L§11.3) and raises. Packages (directories,
L§11.1) are the host's concern: the engine passes the dotted path through to the
resolver, which maps it to a source.

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
  | Faulted(fault)                    # a limit, cancellation, or internal fault

PauseReason = Step | Breakpoint(id) | RaiseTrap | HostPause
EngineFault = LimitExceeded(kind) | Cancelled | Internal
```

### 7.3 Driving and resolving

The host advances an instance with two conceptual operations (their exact factoring
is binding-specific, §12):

```
run(instance, directive) -> Outcome      # start, or continue after a Paused
resolve(instance, resolution) -> Outcome # continue after a Suspended
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

### 7.4 Safe points

A **safe point** is a location where the engine may stop with fully inspectable and
resumable state. The engine must place safe points at least:

- between statements (L§7), and
- at call entry and at return.

These are the points at which the engine evaluates stop conditions (breakpoints,
the current step target, a host pause request, and resource limits, §10). Between
safe points the engine's internal state need not be inspectable. The default
observation granularity is per-statement (§8.1); an instance may be configured for
finer, per-subexpression safe points at additional cost.

### 7.5 Capability requests

A `Suspended` outcome carries a **capability request**: the capability's identity
(which registered capability, §5.3) and its bound arguments (as handles). The host
fulfills it and resolves with `Value(h)` — which becomes the capability call's
result — or `Raise(h)` — which raises `h` at the call site. A `to` capability is
resolved with a `Value` of `nil` or an equivalent no-value resolution. The
capability identity must be stable across runs so that resolutions can be recorded
and replayed (§11).

### 7.6 Reentrant drives

A synchronous foreign function (§5.2) may drive the instance reentrantly by invoking
a Doodle callable. The nested drive returns its own outcome to the host; a nested
`Suspended`/`Paused` propagates outward as the host chooses to handle it. Reentrant
drives share the instance's single heap stack and its instrumentation.

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
driving in a `Step*` directive.

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
set_breakpoint(instance, module_id, line) -> id
clear_breakpoint(instance, id)
```

Under a `Continue` or `Step*` directive the engine stops with `Paused(Breakpoint(id))`
at the first safe point at or after a breakpointed position. Under `RunToCompletion`
breakpoints are ignored. Conditional breakpoints (a Doodle predicate evaluated at the
safe point) are a noted extension (Appendix B), since evaluating a condition runs
Doodle code.

### 8.7 Trap on raise

The host may enable **raise-trapping**. When enabled, the engine stops with
`Paused(RaiseTrap)` at the point an exception is raised (L§12.1), *before* the stack
unwinds, so the debugger can inspect the raising frame with the stack intact. Resuming
continues the unwind (the exception propagates normally). Raise-trapping is independent
of whether the exception is later caught.

### 8.8 Host-requested pause and observation mode

The host may request a pause; the engine stops with `Paused(HostPause)` at the next
safe point. An instance's observation mode (per-statement vs. per-subexpression safe
points, and whether local-binding capture is eager) is set in `config` and may be
adjustable, letting a host trade fidelity for speed — e.g. run `RunToCompletion` with
coarse safe points when nobody is watching, and switch to fine stepping when the user
opens the debugger.

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
- An uncaught exception that reaches a drive boundary yields `Raised(exception, trace)`.
  The **trace** is captured at `raise` (L§12.1) and includes the live frames and the
  bounded tail-elided history (§8.3) as positions and callables.
- **Driving is inherently protected.** Every `run`/`resolve` returns an outcome; a
  Doodle error never propagates as a host-language exception or crash. (This subsumes
  the role of a `pcall`-style protected-call primitive.)
- **Engine faults** — resource-limit exhaustion, cancellation, or an internal
  invariant violation — are `Faulted`, distinct from Doodle exceptions and **not**
  catchable by Doodle code.
- A foreign function that fails in a way the program should handle raises a Doodle
  exception; a host-level failure it cannot express becomes an engine fault surfaced
  to the driving host.

---

## 10. Control, limits, and concurrency

### 10.1 Cancellation

The host may request **cancellation** of a running or suspended instance. At the next
safe point the engine unwinds the stack — restoring dynamic-parameter bindings and
running block-protocol cleanup, as for an exception — and returns `Faulted(Cancelled)`.
Cancellation is **not** catchable by Doodle code. This is the "stop button."

### 10.2 Limits

An instance is configured (§3.1) with limits appropriate to an untrusted or
kid-authored program:

- a **step budget** (safe points executed) — since the engine owns no clock, wall-clock
  timeouts are enforced by the host via the step budget or by cancelling;
- a **heap limit** (bytes/objects);
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
order, each capability request's identity and the resolution supplied.

**Replay.** To replay, the host creates a fresh instance, loads the same program, drives
it, and — instead of performing effects — supplies the recorded resolutions in order for
each `Suspended`. The reconstruction is identical: same positions, same stack, same
values. This yields deterministic forward reconstruction to any step (fast-forward),
and shareable replay artifacts.

**Requirements on the engine.** For replay to hold, the engine must expose stable
capability-request identities (§7.5) and must have no hidden nondeterminism (§2):
value identity, GC, and hashing must not leak observable order into Doodle; in
particular **dict iteration order must be deterministic** (see Appendix B — this is a
constraint that feeds back into L§4.8).

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
