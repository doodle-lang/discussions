# The Doodle Language — An Overview

**Status:** Informative, not normative — the
[language specification](spec/language.md) is authoritative, and this
overview tracks it (including the Appendix C resolutions of the
[implementation plan](plan/implementation.md)) as of 2026-07-17. Doodle is
under active development; the front end exists, the execution engine is in
progress, and everything here describes the language as specified.

---

## 1. What Doodle is

Doodle is a small, dynamically typed programming language for children —
a modern successor to Logo — built around an uncomfortable observation:
most languages designed *for* kids are languages kids must eventually
*leave*. Their friendliness is a ceiling. Doodle's goal is the opposite: a
language a ten-year-old can start in and a seventeen-year-old need not
abandon, because the simple surface sits on a real language underneath.

A few principles shape everything:

- **No magic boundaries.** Anything the standard library does should be
  expressible in ordinary Doodle. `repeat` is a library function, not a
  keyword. The turtle is a library, not a built-in. When a kid asks "could
  I have written that myself?", the answer should be yes — and the source
  is right there to read.
- **Explicit over implicit.** There is no truthiness, no operator
  overloading, no optional-chaining shortcuts, no clever coercions. When a
  shorthand would hide what's happening, Doodle omits the shorthand.
- **Loud failure.** Using a value that doesn't exist, indexing past the
  end, treating nothing as something — these fail immediately, at the
  point of the mistake, with an error message that names the value, points
  at the code, and suggests the fix. Error messages are a designed
  surface, not an afterthought.
- **Robust to inexperience.** Features that are safe only in disciplined
  hands didn't make the cut. The language should be hard to hold wrong.

And one architectural promise that shapes the whole experience: Doodle
programs run on a **deterministic, resumable engine**. Execution can be
paused between any two statements, inspected, stepped, and — because every
effect flows through a recordable boundary — *replayed exactly*. Watching
your program run, line by line, is not a debugger bolted on later; it is
how the engine works.

## 2. A first program

```doodle
import turtle.*

repeat(4) do
    forward(50)
    right(90)
end
```

A square, drawn by a turtle you can watch move. Three things are already
visible. Function calls always use parentheses — `forward(50)`, never
`forward 50` — one rule with no exceptions. The `do … end` construction is
a **block**: a chunk of code handed to `repeat`, which runs it four times.
And `repeat` itself is not a keyword; it is an ordinary function from the
standard library that happens to take a count and a block. You could write
it yourself, and one day you probably will.

## 3. The shape of a program

Statements end at the end of a line. A semicolon lets you put two on one
line; a line ending in a binary operator or a comma continues onto the
next:

```doodle
let total = price +
    shipping +
    tax
```

(The `+` stays at the end of the line — a trailing operator continues the
statement; a leading one never joins backward.)

Comments run from `#` to the end of the line. There are no block comments.

Names are case-sensitive and fully Unicode: `angle`, `角度`, `θ`, and
`café` are all fine identifiers, and the two ways of typing `café`
(composed or decomposed) are the *same* identifier, because source text is
normalized before anything else looks at it.

## 4. Values

Doodle has a deliberately small set of value kinds, each doing one job.

**Numbers.** Integers are arbitrary precision — `2 ** 100` is just a big
number, never an overflow. Floats are IEEE doubles. Division always gives
you the real answer: `5 / 2` is `2.5`. When you want integer division you
say so: `5 // 2` is `2`, and `5 % 2` is `1`. Mixed comparisons do what
math says: `1 == 1.0` is `true`.

**Booleans — strictly.** `true` and `false` are the only truths. An `if`
condition must actually *be* a Boolean: `if 0 then …` and
`if list then …` are errors, not clever idioms. "Is this true?" and "does
this exist?" are different questions, and Doodle keeps them different.

**`nil`** is the single value meaning "nothing here." There are no `?.` or
`??` operators to quietly pass it along; if `nil` reaches an operation
that needs a value, that's a loud error at the spot where the assumption
broke. Handling `nil` is ordinary code: `if x == nil then … end`.

**Strings** are immutable Unicode text, and a "character" is what a human
would call a character:

```doodle
length("café")        # 4 — however the é was typed
length("👨‍👩‍👧‍👦")         # 1 — one family, one character
"café" == "café"      # true, composed or decomposed
```

Strings interpolate expressions with braces:

```doodle
print("You rolled {die1 + die2}!")
```

**Lists and dicts** are the mutable workhorse collections:

```doodle
let scores = [10, 25, 3]
scores[0]                       # 10 — zero-based; out of range is an error
let ages = {alice: 9, bo: 10}   # bare-word keys are strings
ages["alice"]                   # 9 — a missing key is an error
```

Dicts remember insertion order — iteration is predictable, always. There
is no slicing, no negative indexing, no ranges: `scores[length(scores) - 1]`
spells out what `scores[-1]` would hide, and that's on purpose.

**Records** are named bundles of fields — data, with no attached methods:

```doodle
record Point with x, y end

let home = Point(x: 0, y: 0)
print(home.x)
```

More on records — including the one genuinely deep choice about them —
in §12.

## 5. Names: `let`, `const`, and what assignment can't do

```doodle
let score = 0        # a new, mutable binding
score = score + 10   # reassignment — the name must already exist
const max_lives = 3  # can never be reassigned
```

Introducing a name and changing one are different acts with different
spellings, so a typo like `scroe = 10` is an error ("no such name — did
you mean `score`?") instead of a silent new variable. Assignment is a
statement, not an expression — `if x = y` is impossible to write.

One more rule with a long shadow: names bound by *declarations* — a `to`,
`fn`, `record`, `protocol`, or `parameter` — can never be assigned over.
`greet = 42` after declaring `to greet()` is a static error. If you want a
mutable binding that holds a callable, say so: `let g = greet`.

## 6. Doing versus computing: `to` and `fn`

Doodle splits callables in two, because kids already think of them as two
things:

```doodle
to draw_square(size)
    "Draws a square with the given side length."
    repeat(4) do
        forward(size)
        right(90)
    end
end

fn area_of_square(size)
    size * size
end
```

A **procedure** (`to`) *does* something and yields no value. A
**function** (`fn`) *computes* something: the value of the last expression
in its body is its result — no `return` ceremony for the common case
(`return expr` exists for early exits).

This split earns its keep at every use site. Producing "no value" is
always fine — a procedure call as its own statement is normal code — but
*using* a no-value is an error exactly where the use happens:

```doodle
let x = draw_square(50)   # error: draw_square is a procedure and
                          # produces no value — did you mean an fn?
```

There is no `None` quietly flowing around, no accidental return values
Ruby-style. And a function must produce a value on every path — a body
that can "fall off the end" without one is rejected when the program
loads, not discovered mid-run:

```doodle
fn double(x)
    let y = x * 2      # error at load: `double` ends without producing
end                    # a value — did you mean to write `y` last?
```

The first string in a body is its **docstring** — documentation the
language keeps and tools can show — with one carefully chosen exception:
if the string is the *entire* body of an `fn`, it's the result, because

```doodle
fn greeting() "hello" end
```

should mean exactly what it looks like it means.

## 7. Control flow

`if` chains with `else if` and closes with a single `end`. It is also an
expression when every branch produces a value:

```doodle
let label = if n > 0 then "positive"
            else if n < 0 then "negative"
            else "zero"
            end
```

Loops come in exactly two keywords. `while` checks first; `loop` runs
until you leave:

```doodle
loop do
    let guess = ask("Your guess?")
    if guess == secret then break end
    print("Nope — try again!")
end
```

There is no `for`, no `until`, no `unless`, no `do…while` — iteration over
collections belongs to library functions and blocks (§8), and everything
else is `while`, or `loop` with `break`. Comparisons don't chain:
`1 < x < 10` is a load error with a hint to write `1 < x and x < 10`.

Three words leave things, at three sizes: **`continue`** ends the current
block iteration, **`break`** leaves the enclosing loop (or block-taking
call), **`return`** leaves the whole procedure or function — from anywhere
inside it, even from deep inside blocks.

## 8. Blocks: the workhorse

A block is code you hand to a call, and it is how almost every control
pattern beyond `if`/`while` works:

```doodle
let total = 0
each(scores) do(s)
    total = total + s
end
```

Blocks see and can change the variables around them — that accumulation
idiom is day-one Doodle. A block takes parameters in parentheses after
`do`, yields the value of its last expression (which `map` collects and
`each` ignores), and answers to the three exits: `continue` skips to the
next iteration, `break` stops `each` entirely, `return` exits the
enclosing function *through* the block.

The crucial limitation is deliberate: **blocks are not values.** You can't
store one in a variable or return one — a block lives and dies with its
call. That keeps `return` meaning what it says. When you want callable
code *as a value*, you want a function value:

## 9. Function values and closures

```doodle
fn make_counter()
    let n = 0
    fn()
        n = n + 1
        n
    end
end

let tick = make_counter()
tick()    # 1
tick()    # 2
```

An anonymous `fn` is a first-class value that **closes over its
surroundings by reference**: it can read and *change* the variables it
captured, and those variables live as long as anything still needs them.
Two closures capturing the same variable share it — which is exactly what
makes a counter a counter. (Inside a function value, `return` is local to
it; only blocks punch outward.)

And the classic trap isn't here: variables in a loop body are fresh each
time around, so closures made in a loop each capture their *own* `i`, not
one shared final value.

## 10. Recursion is a first-class loop

Doodle guarantees **proper tail calls**: a function whose last act is
calling (itself, or anything) reuses its stack frame. Recursion isn't a
stack-overflow trap; it's a legitimate way to loop forever:

```doodle
to spiral(size)
    if size > 200 then return end
    forward(size)
    right(91)
    spiral(size + 2)
end
```

The engine keeps a bounded history of those reused frames, so the
debugger can still tell you "`spiral` has called itself 47 times" while
using constant memory. Watching a recursive turtle program draw, with the
call counter ticking, is one of the experiences Doodle is built around.

## 11. Context without plumbing: dynamic parameters

How does `forward` know what color to draw? Not a global you overwrite,
and not a parameter you thread through every call — a **dynamic
parameter**, set for the duration of a region of your program:

```doodle
parameter pen_color = "black"

with pen_color = "red" do
    draw_square(50)          # everything inside draws red…
end
draw_square(50)              # …and this one is black again
```

`with` sets the value for everything that runs inside it — including
functions called from other modules — and restores the old value on the
way out *no matter how* you leave: normal completion, `return`, `break`,
or an error. This is the mechanism behind pen color, drawing targets
(draw to the screen, to an image, to a list of commands you can replay),
animation speed, and any context of your own invention. It's ordinary and
declared, not magic: `parameter` is right there in the source.

## 12. Records, and the one deep choice

Records hold data. Operations on records are plain functions — there are
no methods, no `self`, no inheritance:

```doodle
record Point with x, y end

fn distance(a, b)
    sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2)
end
```

The deep choice: is a record a *value* (copied on assignment, like a
number) or a *reference* (shared, like a list)? Doodle makes you say,
because both are real:

```doodle
record Point with x, y end          # value: assignment copies

ref record Turtle with x, y, heading end   # reference: assignment shares

let a = Point(x: 1, y: 2)
let b = a
b.x = 99          # a.x is still 1 — b was a copy

let t = Turtle(x: 0, y: 0, heading: 0)
let u = t
u.heading = 90    # t.heading is 90 too — one turtle, two names
```

A `Point` is data; a `Turtle` is a *thing with identity*. The keyword at
the declaration tells every reader which they're holding. Equality is
structural either way — two `Point(x: 1, y: 2)`s are `==` — and nominal:
a `Point` never equals a `Coordinate`, even with identical fields.

## 13. Protocols: how your types join the language

Polymorphism comes from **protocols** — named sets of signatures a type
can implement. This is how user types become iterable, printable, and
usable exactly like built-ins:

```doodle
protocol Iterable
    """Types whose elements can be visited in order."""
    to each(self, do body)
        "Visits each element in order."
    end                       # empty body: each is *required*
end
```

A member with an empty body (a docstring is fine) is **required**; a
member with a body is a **default** that implementors inherit. Every
member closes with its own `end` — the same rule as every `to` and `fn`
everywhere.

Now the payoff. Ranges aren't built into Doodle — here is the *entire*
mechanism for adding them:

```doodle
record Range with start, stop end

fn range(start, stop)
    Range(start: start, stop: stop)
end

implement Iterable for Range
    to each(r, do body)
        let i = r.start
        while i < r.stop do
            body(i)
            i = i + 1
        end
    end
end

each(range(1, 5)) do(i)
    print(i)                 # 1 2 3 4
end
```

Your `Range` now works with `each`, `map`, `filter` — everything built on
`Iterable` — through the same door the built-in types use. Calls dispatch
on the first argument's type; `x is Iterable` asks whether a type
implements a protocol; and if two protocols ever collide on a name, you
qualify: `Iterable.each(thing) do … end`.

What protocols deliberately do *not* enable: operator overloading. `+`
means numeric addition or string concatenation, ever. A vector library
says `add(v, w)` — two characters longer and never a mystery.

## 14. Strings, properly

Interpolation takes any expression — but stays on one line, can't be
empty, and can't hide a comment; each of those is a specific, friendly
error rather than a mysteriously broken string. Escapes are a closed set
(`\n`, `\t`, `\"`, `\\`, `\0`, `\xHH`, `\u{…}`): an unknown escape like
`\q` is an error naming it, not a silent `q`.

Multi-line text uses triple quotes, with indentation handled the way you
already indented it:

```doodle
let recipe = """
    Ingredients:
      - flour
      - sugar
    Mix well.
    """
```

The closing `"""`'s indentation sets the margin; every line must match it
*exactly* (tabs and spaces are not interchangeable — the error will tell
you which character differs); what's beyond the margin, including the
extra two spaces before `- flour`, is your text. A one-liner
`"""He said "hi"."""` also works — handy for docstrings and for text
containing quotes.

Raw bytes are a different type with a different literal — `b"GET /"` —
because text-with-an-encoding and bytes-you-must-not-touch are different
things, and Doodle never blurs them.

## 15. Programs made of modules

Every file is a module; every directory of files is a package. Imports
come in exactly one keyword with three shapes:

```doodle
import math                  # qualified: math.sqrt(2)
import math.sqrt             # one name, directly: sqrt(2)
import turtle.*              # everything turtle exports, unqualified
```

The wildcard is how a program declares its domain — a turtle program
opens with `import turtle.*` and speaks turtle natively. It's also
honest: that line is *all* the magic there is, and replacing it with
`import music.*` puts you in a different world by the same mechanism.
Modules load once (their top level runs a single time, lazily), cycles
are an error with a map of the cycle, and a module's `exports` list is
its public face. Imported names are live, read-only views of the
exporter's bindings — and `with pen_color = …` in *your* file genuinely
affects the turtle library's drawing, because a dynamic parameter is the
same parameter everywhere it's visible.

## 16. When things go wrong

Expected failures are **values**. Parsing user input can fail, so it
returns `nil` and you check:

```doodle
let n = to_integer(answer)
if n == nil then
    print("That's not a number — try again!")
end
```

Exceptions exist for the genuinely exceptional, with the barest possible
machinery — `raise`, and `try`/`rescue`:

```doodle
try
    play_level(current)
rescue e
    if e is SaveFileError then
        print("Couldn't load your save: {e.message}")
    else
        raise            # not ours — pass it on, trace intact
    end
end
```

There are no exception hierarchies, no `finally` (cleanup belongs to
`with`-style helpers, which restore on *any* exit), and — by design — most
programs never write a `rescue` at all. An uncaught error stops the
program with a stack trace captured at the `raise`, tail-call history
included.

## 17. What Doodle deliberately leaves out

The cuts are as designed as the features. Missing on purpose:

| Not in Doodle | Instead |
|---|---|
| truthiness | real Booleans, asked explicitly |
| `?.` / `??` | `if x == nil` — visible handling |
| operator overloading | named functions (`add(v, w)`) |
| classes, methods, inheritance | records + functions + protocols |
| `for`, `until`, `unless`, ternary | `while`, `loop`, blocks, `if` expressions |
| ranges, slicing, negative indices | explicit arithmetic, library functions |
| chained comparisons | `a < b and b < c` |
| line-joining `\`, block comments | real lines, `#` per line |
| implicit conversions | one widening (int→float), everything else spelled out |
| threads, async | a deterministic single-threaded core; effects live at the host boundary |

Every one of these is a place where a shorthand would have made programs
shorter and understanding longer. Doodle consistently sells conciseness
to buy legibility — because its programs are read by learners, teachers,
and parents, not just written by experts.

## 18. Under the hood, briefly

The engine that runs Doodle is a **resumable state machine**: it advances
one observable step at a time and can stop between any two statements
with its entire call stack — locals, dynamic bindings, tail-call
counters — open for inspection. Every effect (drawing, input, time,
randomness) is a *capability* provided by the host program, which is what
makes three promises possible:

- **It runs anywhere** — including a browser tab, animating a turtle on
  the main thread without freezing the page, stop button always live.
- **Execution is visible by default.** The line highlight, the call
  stack, "this function called itself 47 times" — these read directly
  from the machine, not from a special debug build.
- **Runs can be replayed exactly.** Record what crossed the boundary and
  you can reconstruct the whole run — the foundation for time-travel
  debugging and for sharing a *run*, not just a program.

The same design shapes interactive use: a future REPL session is a module
built up incrementally, where re-declaring `greet` *replaces* it — every
existing caller picks up the fix — while inside any single entry the
normal rules hold exactly.

## 19. Where things stand

The language and engine specifications are drafted and continuously
reconciled against the implementation (front end largely built; the
machine core is next), with every discovered ambiguity resolved in the
spec before the code that depends on it lands. The standard library —
including the turtle — is specified as an ordinary set of modules,
because that is what it is.

To go deeper:

- [`spec/language.md`](spec/language.md) — the language, normatively
- [`spec/engine.md`](spec/engine.md) — the embedding and debugging API
- [`plan/implementation.md`](plan/implementation.md) — architecture,
  milestones, and the running log of design decisions
- [`Claude-Doodle language discussion.md`](<Claude-Doodle language discussion.md>)
  — the original design conversation, where the rationale lives
