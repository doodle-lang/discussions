# The Doodle Programming Language — Language Specification

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-07-04

This document specifies the Doodle programming language: its lexical
structure, syntax, type system, and evaluation semantics. It is derived from
the design discussion archived alongside it
([`../Claude-Doodle language discussion.md`](<../Claude-Doodle language discussion.md>)),
which records the rationale behind most decisions here. Where the discussion left a
question genuinely open, or where writing a precise rule forced a choice, this
specification makes a call and records it in
[Appendix D](#appendix-d-specification-decisions-and-open-issues).

---

## 1. Introduction

Doodle is a small, dynamically typed, lexically scoped language designed as a
first language for children that remains a real language they need not outgrow.
Its surface is deliberately explicit — it favors clarity over conciseness — and
its core is small, with a guiding principle that almost everything a program can
use should be expressible *in Doodle itself*.

### 1.1 Scope

This specification covers:

- lexical structure (§3),
- the type system and value model (§4),
- bindings, scope, and dynamic parameters (§5),
- expressions (§6) and statements (§7),
- procedures, functions, and blocks (§8),
- records (§9), protocols (§10), and modules (§11),
- errors (§12), reflection (§13), and the execution model (§14),
- the normative interface the language expects the standard library to provide (§15).

### 1.2 What this specification does not cover

**The standard library is out of scope.** It is a separate, adjacent
specification to be written later. In particular, this document does **not**
define any of the following, all of which belong to the standard library:

- named values and functions such as `print`, `read_line`, `each`, `map`,
  `filter`, `reduce`, `length`, `append`, `sqrt`, `to_string`, `assert`, `copy`,
  `is_same`, `type_of`, and the like;
- the concrete protocols `Stringable`, `Hashable`, `Iterable`, and `Lengthable`,
  together with their default implementations for built-in types;
- domain libraries such as `turtle`, `collections`, `math`, `random`, `binary`,
  and `bits`;
- the *prelude* (the set of names implicitly available in every program) and any
  editor-supplied preambles;
- the interactive environment (REPL, canvas, debugger, animation) and the
  embedding/host interface.

The language proper defines keywords, operators with fixed meaning, literal
syntax, and the four structural facilities (records, procedures, modules,
protocols). Everything the language semantics unavoidably rely on from the
standard library is enumerated as a small set of *hooks* in §15.

### 1.3 Design principles

These principles are informative but explain many rules below.

1. **No magic boundaries.** Anything the standard library does should be
   expressible in ordinary Doodle. The language provides mechanisms; libraries
   provide policy. Control constructs like iteration are library functions, not
   keywords, wherever that is possible.
2. **Explicit over implicit.** When a shorthand would hide the operation it
   performs, the language omits it. There is no truthiness, no `nil`-propagation
   operators, no operator overloading, no chained comparison, no slicing, and no
   implicit numeric coercion beyond int→float widening.
3. **Robust to inexperience.** Features that require expert discipline to use
   safely are omitted rather than provided in a diminished form.
4. **Loud failure.** Operations on absent or malformed data raise errors at the
   point of failure rather than silently producing a value.
5. **One way, uniform.** Function calls always use parentheses; behavior is
   always a call, never method syntax on data; a name always denotes a value.

### 1.4 Terminology and conformance (informative)

The key words **must**, **must not**, **should**, and **may** are used with
their ordinary normative sense. "Raises an error" means a Doodle exception is
raised (§12); an *uncaught* error terminates the program with a diagnostic. A
"static error" is detected before execution (by a conforming
compiler/loader/tool) and prevents the offending unit from running; the language
is dynamically typed, so most errors are dynamic.

An implementation is conforming if it accepts the programs this document
describes as valid and evaluates them with the semantics given here, and rejects
programs this document describes as ill-formed.

---

## 2. Grammar notation

Grammar fragments use the following EBNF conventions:

| Notation | Meaning |
|---|---|
| `'text'` | a literal terminal |
| `UPPER` | a lexical (token) nonterminal, defined in §3 |
| `lower` | a syntactic nonterminal |
| `A B` | `A` followed by `B` |
| `A \| B` | `A` or `B` |
| `( … )` | grouping |
| `A?` | zero or one `A` |
| `A*` | zero or more `A` |
| `A+` | one or more `A` |
| `A sep-by ','` | one or more `A` separated by `,` (a trailing separator is noted where allowed) |

Newlines are significant at the statement level (§3.2); within the grammar,
`NEWLINE` denotes a statement terminator. A consolidated grammar appears in
[Appendix A](#appendix-a-grammar-summary).

---

## 3. Lexical structure

### 3.1 Source text and encoding

A Doodle source file is a sequence of Unicode code points encoded as UTF-8.
Before tokenization, source text is normalized to Unicode Normalization Form C
(NFC); consequently two identifiers that are canonically equivalent are the same
identifier. Line endings are also canonicalized before tokenization: a CRLF (a
carriage return immediately followed by a line feed, §3.2) is replaced by a
single line feed, so the tokenizer and source positions see line-feed-only text.

**Source positions.** A source position is defined over the NFC-normalized text
above, not over the bytes as written on disk. A position is a 1-based line
(§3.2) and a 1-based column, where the column counts **Unicode code points** from
the start of the line (the first code point of a line is column 1). A span is a
range of source text between two positions, measured in code points — not in the
grapheme-cluster "characters" of §4.4. Code points are the position unit the
language and the engine's observation interface (E§8.1) report; a binding or host
may convert them to another unit (bytes, UTF-16 code units, or grapheme
clusters) for display or editor integration.

Doodle is **case-sensitive**: `Point` and `point` are distinct.

The file name extension is `.doodle`. A file whose name matches `*_test.doodle`
is, by convention, a test file; this convention is not part of the language
semantics.

### 3.2 Lines, statements, and continuation

The primary statement separator is the end of a line (a line feed, optionally
preceded by a carriage return). A semicolon `;` is also a statement separator
and may be used to place several statements on one line. Consecutive separators
and blank lines are permitted and have no effect.

A statement continues onto the following physical line, without an explicit
continuation marker, when the current line:

- ends inside an unclosed `(`, `[`, or `{`; or
- ends with a **continuation-trigger token**: one of the binary operators `+`,
  `-`, `*`, `/`, `//`, `%`, `**`, `==`, `!=`, `<`, `>`, `<=`, `>=`, the word
  operators `and`, `or`, `is`, or a comma `,`.

A trailing `-` or `+` continues the line even though the same token also begins a
unary expression: whether it is unary or binary is a parsing question, so
continuation keys on the token, not on its role. The assignment `=` is **not** a
continuation trigger (a line ending in `=` terminates, so an incomplete `let x =`
is reported at the `=`), and neither is the unary `not`, the access `.`, nor the
punctuation `:`. When finding a line's last token, trailing whitespace and a
trailing comment (§3.3) are ignored — a comment between an operator and the
newline does not break continuation. Only a *trailing* trigger continues; a
leading operator never joins to the previous line.

There is no automatic insertion of separators in any other circumstance; because
blocks are delimited by keywords and `end` rather than by layout, no
JavaScript-style ambiguities arise.

Indentation is **not** syntactically significant. It is a matter of style; the
conventional indent is four spaces (§Appendix D).

### 3.3 Comments

A comment begins with `#` and extends to the end of the line. There are no
block/multi-line comments. A `#` on the first line (e.g. `#!/usr/bin/env
doodle`) is an ordinary comment, so shebang lines are permitted.

### 3.4 Identifiers

Identifiers follow Unicode Standard Annex #31 (UAX #31) with `_` permitted as a
start and continue character:

```
IDENT       = XID_START XID_CONTINUE*
XID_START    = '_' | <any character with the Unicode XID_Start property>
XID_CONTINUE = '_' | <any character with the Unicode XID_Continue property>
```

The `XID` properties (UAX #31's recommended default) are used rather than the
plain `ID` properties because they are closed under NFC: since source is
NFC-normalized (§3.1) and identifiers are compared by NFC-normalized code-point
equality, an `XID` identifier stays a valid identifier after normalization,
whereas an `ID` one need not. Non-ASCII
identifiers are fully supported (e.g. `角度`, `длина`, `θ`, `café`). Emoji and
other characters outside UAX #31 are not permitted in identifiers.

**Module names** are more restricted, because they map to file-system paths: a
module name must match `[a-z][a-z0-9_]*` (lowercase ASCII letters, digits, and
underscores, beginning with a letter). See §11.

A keyword (§3.5) is not an identifier and may not be used as one.

### 3.5 Keywords

The following words are reserved and may not be used as identifiers:

```
and       as        break     const     continue  do        else
end       exports   extends   false     fn        for       if
implement import    is        let       loop      module    next
nil       not       or        parameter protocol  raise     record
ref       rescue    return    then      to        true      try
use       while     with
```

Notes:

- `true`, `false`, and `nil` are reserved words that also serve as literals
  (§3.6).
- `parameter` reserves the dynamic-parameter declaration form (§5.5). It was
  added by this specification; see Appendix D.
- `next` and `use` are reserved but unused by this version. They are held back
  deliberately: `next` because it is a plausible future synonym/alternative to
  `continue`, and `use` because it participated in earlier import syntax.
  Reserving them avoids churn if they are reintroduced. (An implementation may
  treat use of a reserved-but-unused word as a static error.)

All other identifiers — including `self`, `print`, `each`, and every standard
library name — are ordinary identifiers, not keywords.

### 3.6 Literals

#### 3.6.1 Integer literals

Integers are arbitrary precision. A literal is written in base 10, 16, 2, or 8.
Underscores may appear between digits (not leading, trailing, or adjacent to a
base prefix) and are ignored.

```
INT     = DEC | HEX | BIN | OCT
DEC     = DIGIT ('_'? DIGIT)*
HEX     = '0x' HEXDIGIT ('_'? HEXDIGIT)*
BIN     = '0b' ('0' | '1') ('_'? ('0' | '1'))*
OCT     = '0o' OCTDIGIT ('_'? OCTDIGIT)*
```

An underscore must fall between two digits: a leading, trailing, doubled, or
prefix-adjacent underscore (`_1`, `1_`, `1__0`, `0x_F`) is a static error. A base
prefix must be followed by at least one valid digit; an empty prefix (`0x`) or a
digit not valid for the base (`0xG`) is a static error. Base prefixes are
lowercase, so `0XFF` is not an integer literal (it lexes as `0` followed by the
identifier `XFF`). A base-10 literal may begin with `0` (`0123` is 123; there is
no C-style octal).

Examples: `42`, `1_000_000`, `0xFF`, `0xdead_beef`, `0b1010_0101`, `0o755`.
Hexadecimal letters are case-insensitive. A leading `-` is the unary minus
operator (§6.5), not part of the literal.

#### 3.6.2 Float literals

```
FLOAT   = DIGITS '.' DIGITS EXP?
        | DIGITS EXP
EXP     = ('e' | 'E') ('+' | '-')? DIGITS
DIGITS  = DIGIT ('_'? DIGIT)*
```

Examples: `3.14`, `2.0`, `1e6`, `1.5e-3`, `3.141_592`. A float has both an
integer and fractional part around the point (there is no `.5` or `2.` form); a
digit is required on each side of `.`. An exponent requires at least one digit
(`1e` and `1e+` are static errors), and the between-digits underscore rule of
§3.6.1 applies. Floats are IEEE-754 binary64.

#### 3.6.3 String literals

A string literal is delimited by double quotes and denotes an immutable Unicode
string.

```
STRING  = '"' ( CHAR | ESCAPE | INTERP )* '"'
```

- `CHAR` is any code point other than `"`, `\`, an unescaped `{`, or a line
  terminator. (A string literal does not span lines; use a triple-quoted string,
  §3.6.4.)
- `ESCAPE` is one of a **closed** set: `\"`, `\\`, `\n`, `\t`, `\r`, `\0`,
  `\xHH` (exactly two hex digits), `\u{H…}` (1–6 hex digits naming a Unicode
  scalar value). A backslash followed by anything else — e.g. `\q` — is a static
  error that names the unknown escape (write `\\` for a literal backslash), and a
  *malformed* form of a known escape (`\x` with fewer than two hex digits, `\u`
  without braces, an empty `\u{}`, or a `\u{…}` with more than six digits) is a
  distinct "malformed escape" error. `\xHH` denotes the code point `U+00HH`
  (`0..FF`), so `"\xE9"` and `"\u{E9}"` denote the same string. A `\u{…}`
  escape must denote a scalar value in `0..D7FF` or `E000..10FFFF`; a surrogate
  code point (`D800..DFFF`) is a static error. Unassigned code points,
  noncharacters, and lone combining marks are permitted — e.g. `"\u{301}"` (a
  bare combining acute accent) is a valid one-grapheme string.
- `INTERP` is `{ expression }` (§6.7). To include a literal brace, write `{{`
  for `{` and `}}` for `}`. The expression may not contain a line terminator
  (§6.7), and an empty interpolation `{}` — or whitespace-only `{ }` — is a
  static error.

The value denoted by a string literal is NFC-normalized (§4.4), so the code
points of the resulting string may differ from those written in the source.

Examples:

```
"hello"
"tab:\there"
"café \u{2728}"
"Hello, {name}! You have {count} messages."
"a literal brace: {{ and }}"
```

#### 3.6.4 Triple-quoted (multi-line) string literals

A triple-quoted string spans multiple lines and applies indentation stripping.
It supports the same escapes and interpolation as ordinary strings.

```
"""
content line
content line
"""
```

Rules:

- The opening `"""` must be the **last token on its line** — not even a comment
  may follow it; the contents begin on the next line. The closing `"""` must be
  the **first token on its line**, apart from the leading whitespace that forms
  the margin; ordinary code may follow the closing `"""`. A `"""` in the middle
  of a content line is literal text, and a content line that must *begin* with a
  literal `"""` is written `\"""`.
- The **margin** is the exact whitespace *string* preceding the closing `"""`,
  matched **byte-for-byte** as a prefix of each content line — there is no
  tab-width or column arithmetic anywhere. It is removed from the start of every
  content line. A content line whose start does not match the margin is a static
  error that names the first differing character (e.g. "a tab where the margin
  has spaces").
- A **truly empty** content line (zero characters) is exempt from the margin
  check and contributes an empty line. A **whitespace-only but nonempty** line is
  an ordinary content line: it must carry the full margin, and its remainder is
  content — so `margin` + two spaces contributes a two-space line. (Editors that
  trim trailing whitespace silently reduce such a line to empty; `\x20` is the
  robust idiom for intentional trailing spaces.)
- Content after the margin is preserved verbatim, **including trailing
  whitespace** (no trailing-whitespace munging).
- Splitting into lines, margin validation, and stripping happen on the **raw
  physical lines**, *before* escape and interpolation processing: a `\n` escape
  does not create a new margin-checked line, and there is **no** backslash-newline
  line continuation.
- The newline immediately after the opening `"""` and the newline immediately
  before the closing `"""` are not part of the value. The **value** is the
  stripped content lines joined with `\n` (zero content lines yield `""`).
- An interpolation `{expr}` still may not contain a line terminator (§6.7); since
  stripping has already run on physical lines, a `{expr}` stays within one
  stripped line and margin stripping never operates on an expression's interior.

Example (margin = 4 spaces):

```
let recipe = """
    Ingredients:
      - flour
      - sugar
    Mix well.
    """
```

yields `"Ingredients:\n  - flour\n  - sugar\nMix well."` (indentation beyond the
margin is preserved).

Triple-quoted strings are the idiomatic form for docstrings (§8.6).

#### 3.6.5 Bytes literals

A bytes literal denotes an immutable sequence of 8-bit values.

```
BYTES   = 'b"' ( BYTECHAR | BYTEESCAPE )* '"'
```

The source between the quotes must be ASCII. `BYTEESCAPE` is the same **closed**
set minus `\u{…}`: `\"`, `\\`, `\n`, `\t`, `\r`, `\0`, and `\xHH` — where `\xHH`
here denotes the byte `0xHH` (`0..FF`), the type's natural unit. The `\u{…}`
escape is **not** valid in a bytes literal (it denotes a character, not a byte)
and is a static error there, as is any unknown or malformed escape.
Interpolation is not performed in bytes literals.

Examples: `b"GET / HTTP/1.1\r\n"`, `b"\x00\x01\x02"`.

Mutable byte buffers and fixed-width integer types are provided by the standard
library (the `binary` module), not the language core.

#### 3.6.6 Boolean and nil literals

```
true    false    nil
```

`true` and `false` are the two Boolean values; `nil` is the sole value of type
`Nil` (§4.6).

### 3.7 Operators and punctuation

The following tokens are operators or punctuation:

```
+   -   *   /   //   %   **
==  !=  <   >   <=   >=
=   .   ,   :   (   )   [   ]   {   }
```

`and`, `or`, `not`, and `is` are spelled as words (§3.5) and act as operators
(§6.5). Their semantics and precedence are given in §6. There are no bitwise
operators, no range operator, no null-coalescing or optional-chaining operators,
and no user-defined operators.

---

## 4. Types and values

Doodle is dynamically typed: values carry their type; variables do not. Every
value has exactly one type. The language provides the following built-in kinds
of value.

### 4.1 Classification

| Category | Types | Mutable? | Assignment/argument semantics |
|---|---|---|---|
| Numbers | `Int`, `Float` | no | by value (immutable) |
| Boolean | `Bool` | no | by value |
| String | `String` | no | by value (immutable) |
| Bytes | `Bytes` | no | by value (immutable) |
| Absence | `Nil` | no | by value |
| List | `List` | yes | by reference |
| Dict | `Dict` | yes | by reference |
| Callable | procedures, functions | no | by reference (identity) |
| Record (value) | user-declared `record` | fields yes | by value (copy) |
| Record (reference) | user-declared `ref record` | fields yes | by reference |
| Module | modules | see §11 | by reference |
| Type / Protocol | type values, protocol values | no | by reference (identity) |

Blocks (§8.5) are **not** values; they are a second-class parameter-passing
mechanism.

### 4.2 Numbers

There are two numeric types:

- **`Int`** — a signed integer of arbitrary precision. There is no overflow and
  no wraparound.
- **`Float`** — an IEEE-754 binary64 floating-point number.

Arithmetic (§6.5): `+`, `-`, `*` on two `Int`s yield an `Int`; if either operand
is a `Float`, the other is widened to `Float` and the result is a `Float`
(int→float widening is the only implicit numeric coercion). Division `/` always
yields a `Float` (e.g. `4 / 2` is `2.0`). Floor division `//` and modulo `%`
require both operands to have the same numeric type after widening and follow
floored semantics. Exponentiation `**` yields a `Float` unless both operands are
`Int` and the exponent is non-negative, in which case it yields an `Int`.

Division or modulo by zero (`/`, `//`, `%`) raises an error. Narrowing a `Float`
to an `Int` never happens implicitly; the standard library provides explicit
conversions.

### 4.3 Boolean

`Bool` has exactly two values, `true` and `false`. Doodle has **no truthiness**:
the condition of `if`, `while`, the operands of `and`, `or`, `not`, and any
place a Boolean is required must be an actual `Bool`. Supplying a non-Boolean
raises an error.

### 4.4 String

`String` is an **immutable** sequence of Unicode text, stored in Normalization
Form C (see below). String literals may interpolate expressions (§6.7).

**A character is an extended grapheme cluster.** The unit the language treats as
one "character" is the user-perceived character — the extended grapheme cluster
of Unicode Annex #29 — not a byte and not a code point. Consequently:

- `length(s)` counts grapheme clusters, so `length("café") == 4` however the
  `é` was composed, `length("🎉") == 1`, and `length("👨‍👩‍👧‍👦") == 1`.
- Iterating a string (`each`) visits one grapheme at a time.
- Indexing `s[i]` (§6.3) is by grapheme cluster and yields a **length-one
  string** (the i-th character a human perceives). There is no distinct
  character type; a character is just a one-grapheme string.

Because grapheme clusters are variable width, `length` is O(n) and `s[i]` is
O(i); prefer iteration (`each`) to index loops for scanning. Grapheme length is
also **not additive** across concatenation — two strings that are each one
grapheme can fuse into a single grapheme when joined (for example, two
regional-indicator characters forming one flag), so `length(a) + length(b)` need
not equal `length(a + b)`.

**Normalization.** Every `String` value is normalized to NFC when it is
constructed — from a literal, from concatenation, or by decoding bytes. This
makes equality *canonical*: two strings that render identically are `==`
regardless of how they were typed (§4.13). It also means a `String` is a
*normalized human-text* value, not a faithful container of arbitrary code
points: the code points of a string may differ from those written or decoded,
and round-tripping text through `String` can change its bytes. Byte- or
code-point-exact work stays in `Bytes` (§4.5); §15 gives the conversions.

**Operations.** Concatenation is `+` (the result is re-normalized to NFC);
repetition is `*` (String × Int). Ordering (`<`, …) is a simple
code-point-lexicographic comparison over the NFC form — total and stable, but
deliberately *not* locale/dictionary collation, which is a standard-library
concern. The full string API (searching, splitting, case mapping, the code-point
and byte views, …) is standard-library, built over the language's Unicode
primitives (§15).

### 4.5 Bytes

`Bytes` is an immutable sequence of 8-bit values, written with a `b"…"` literal
(§3.6.5). It supports indexing (yielding an `Int` in 0–255, O(1)). Richer binary
facilities live in the standard library.

`Bytes` is the **faithful** byte container that complements `String`: the
language provides conversion between a `String` and its NFC UTF-8 `Bytes` (§15).
A string's bytes are always its NFC UTF-8 form; decoding arbitrary `Bytes` to a
`String` validates the UTF-8 (raising on invalid input) and normalizes to NFC,
so `Bytes → String → Bytes` need not reproduce the original bytes. Work that
must preserve exact bytes (wire formats, files that must not be altered) stays in
`Bytes` throughout and never becomes a `String`.

### 4.6 Nil

`Nil` has the single value `nil`, denoting the absence of a meaningful value.
Doodle provides no operators that silently propagate or default `nil`; code that
must handle `nil` does so with ordinary control flow (e.g. `if x == nil then
…`). Most operations applied to `nil` (field access, arithmetic, indexing)
raise an error.

### 4.7 List

`List` is a mutable, dynamically sized, **reference-typed** ordered sequence,
written `[a, b, c]` (a trailing comma is permitted; `[]` is empty). Elements may
be of any type. Indexing is zero-based; `list[i]` requires `0 <= i < length` and
otherwise raises. Negative indices are not supported. List operations (append,
etc.) are standard-library functions.

Because lists are reference-typed, assigning or passing a list shares it;
mutations are visible through every binding that refers to it.

### 4.8 Dict

`Dict` is a mutable, **reference-typed** mapping, written `{k1: v1, k2: v2}` (a
trailing comma is permitted; `{}` is the empty dict — note that `do … end`, not
`{ … }`, denotes a block, so `{}` is unambiguously an empty dict). A bare-word
key in a literal denotes a string key: `{name: "Alice"}` has the string key
`"name"`.

Keys must be hashable (see §15): numbers, booleans, strings, and by default
records are usable as keys; lists and dicts are not valid keys. Indexing
`dict[key]` requires the key to be present and otherwise raises. Iteration and
lookup-with-default are provided by the standard library.

**Iteration order is insertion order.** Iterating a dict (or its keys/values)
visits entries in the order their keys were first inserted; reassigning an
existing key leaves its position unchanged, and removing then reinserting a key
moves it to the end. This order is deterministic and independent of key hashing
or allocation — a property the engine relies on for reproducible execution and
replay (see the engine specification, §11).

### 4.9 Procedures, functions, and blocks

Procedures (declared with `to`) and functions (declared with `fn`, including
anonymous `fn` expressions) are first-class values (§8). They may be bound,
passed as arguments, returned, and stored. Two callable values are equal only if
they are the same callable (identity).

**Blocks** (`do … end`, §8.5) are not values. A block may only be passed as the
trailing block argument of a call and invoked by the receiving callable; it may
not be stored, returned, or passed onward (except by being invoked in tail
position).

### 4.10 Records

A record type is declared with `record` (value semantics) or `ref record`
(reference semantics) (§9). Records are **nominal**: two record types with
identical field names are nonetheless different types, and their instances are
never equal to each other.

- A **value record** is copied on assignment and on argument passing (§4.14).
- A **reference record** is shared on assignment and on argument passing.

In both cases, fields are mutable and are read/written with `.` (§6.2, §7.4).

### 4.11 Modules

A module is a first-class value obtained by importing (§11). Its members are
accessed with `.` (e.g. `math.pi`). A module's set of exported names is fixed
once loaded; a module cannot be mutated from outside (its own mutable
module-level bindings may change from within it).

### 4.12 Types and protocols as values

Each built-in type is denoted by a type value: `Int`, `Float`, `Bool`,
`String`, `Bytes`, `Nil`, `List`, `Dict`, `Procedure`. (`Number` denotes "either
`Int` or `Float`.") A record declaration introduces a type value of the same
name that also serves as the constructor (§9.2). A protocol declaration
introduces a protocol value (§10). Type and protocol values are used with the
`is` operator (§6.5) and with reflection (§13).

> The exact spelling of built-in type values, and whether `Procedure`
> distinguishes procedures from functions, is provisional; see Appendix D.

### 4.13 Equality and identity

The `==` and `!=` operators test **structural equality** and are built into the
language (not user-overridable):

- Two values are `==` only if they have the same type.
- Numbers compare by mathematical value **within** a type after int→float
  widening for mixed comparisons (`1 == 1.0` is `true`); other primitives
  (booleans, bytes, nil) compare by their contents.
- Strings compare by **canonical (NFC) equivalence**: two strings are `==` iff
  they are Unicode-canonically equivalent. Because string values are stored in
  NFC (§4.4), this is just comparison of their normalized contents, so two
  strings that render identically are equal regardless of how they were typed
  (`"café"` precomposed equals `"café"` written as `e` + combining acute). This
  is *canonical*, not *compatibility*, equivalence — `"①" != "1"`, and the `ﬁ`
  ligature is not equal to `"fi"`.
- Lists and dicts compare by structure (same length/keys and pairwise-equal
  elements/values).
- Records compare by type and by structural equality of their fields; this holds
  for both value and reference records. Structural comparison is cycle-safe (a
  conforming implementation must terminate on cyclic reference-record graphs).

**Identity** — whether two references denote the same object — is a distinct
question, answered by a standard-library function (conventionally `is_same`),
not by `==`. Identity is only observable for reference-typed values.

### 4.14 Value vs. reference semantics

The semantics of `=` (binding/assignment) and of binding an argument to a
parameter depend only on the value's kind:

- **Reference-typed** values (lists, dicts, reference records, callables,
  modules, types) are *shared*: the new binding refers to the same object, and
  mutations are visible through all bindings.
- **Value-typed** values (numbers, booleans, strings, bytes, nil, value records)
  are *copied*. For value records, copying is defined recursively: value-typed
  fields are copied, and reference-typed fields are shared (their reference is
  copied, not their target). This mirrors C-style struct copy.

Thus for a value record `Point`:

```
let a = Point(x: 3, y: 4)
let b = a          # b is a copy
b.x = 10           # does not affect a
```

and for a reference record `Turtle`:

```
let t = Turtle(position: home, heading: 0, pen_down: true)
let u = t          # u and t share the same turtle
u.heading = 90     # visible as t.heading == 90
```

---

## 5. Bindings and scope

### 5.1 Lexical scope

Names are resolved lexically: the meaning of a name in a piece of code is
determined by the enclosing textual structure, not by the dynamic call stack.
The one exception is dynamic parameters (§5.5), which are resolved dynamically
and are introduced only by the `parameter`/`with` forms.

Scopes nest. A name introduced in an inner scope shadows the same name in an
outer scope for the remainder of the inner scope. An implementation should warn
when an inner declaration shadows an outer binding, but shadowing is not an
error.

### 5.2 `let` and `const`

```
let-stmt   = 'let' IDENT '=' expression
const-stmt = 'const' IDENT '=' expression
```

`let` introduces a new **mutable** binding in the current scope, initialized to
the value of the expression. `const` introduces a new binding that may not be
reassigned. There is no uninitialized declaration: an initializer is always
required (write `let x = nil` for a deliberate placeholder).

`const` restricts *reassignment of the binding*, not mutation of the value it
refers to: `const xs = []` forbids `xs = …` but permits mutating the list.

Declaring a name that already exists in the *same* scope is a static error.

### 5.3 Assignment

```
assign-stmt = lvalue '=' expression
lvalue      = IDENT
            | expression '.' IDENT
            | expression '[' expression ']'
```

`name = value` reassigns an existing binding; assigning to a name that is not in
scope, or to a `const` binding, is an error (static where determinable).
`target.field = value` mutates a record field; `target[key] = value` mutates a
list element or dict entry.

Assignment is a **statement, not an expression**: it produces no value and may
not appear where an expression is expected. This removes the `if (x = y)` class
of mistakes.

### 5.4 Blocks and per-invocation scope

Every body delimited by a construct (`if`/`while`/`loop`/`with`/`try`, a
function or procedure body, and a `do … end` block argument) is its own scope.
Each *invocation* of a block establishes a fresh scope, so a `let` inside a loop
body or an iterating block introduces a distinct binding on each iteration.

### 5.5 Dynamic parameters

A **dynamic parameter** is a named binding whose value is resolved dynamically:
a reference to it yields the value most recently established by an enclosing
`with` on the current call path, or its declared default if none is in effect.
Dynamic parameters are the language's mechanism for contextual state (for
example, a drawing library's current pen color or target surface).

```
parameter-decl = 'parameter' IDENT '=' expression
with-stmt      = 'with' IDENT '=' expression 'do' body 'end'
```

- `parameter name = default` declares a dynamic parameter at module level and
  gives it a default value. The name must be declared before it is used; reading
  or `with`-binding an undeclared dynamic parameter is an error (there is no
  auto-creation).
- A reference to a declared dynamic parameter name evaluates to its current
  dynamic value. There is no sigil.
- `with name = value do … end` establishes `value` as the current value of the
  dynamic parameter `name` for the dynamic extent of the body, and restores the
  previous value when the body exits **by any means** — normal completion, an
  exception, or a non-local exit (`return`/`break`/`continue`). Restoration on
  exit is guaranteed (§8.7 discusses the tail-call consequence).

`with` binds exactly one dynamic parameter; nest `with` forms to bind several.
A lexical binding and a dynamic parameter are distinct facilities: `with` never
rebinds a `let`/`const` name, and `=` never rebinds a dynamic parameter.

Example:

```
parameter pen_color = "black"

to demo()
    draw()                       # sees pen_color == "black"
    with pen_color = "red" do
        draw()                   # sees pen_color == "red"
    end
    draw()                       # sees "black" again
end
```

---

## 6. Expressions

An expression computes a value. This section defines expression forms and their
precedence.

### 6.1 Primary expressions

```
primary = INT | FLOAT | STRING | TRIPLE_STRING | BYTES
        | 'true' | 'false' | 'nil'
        | IDENT
        | list-literal
        | dict-literal
        | '(' expression ')'
        | anon-fn
```

A bare `IDENT` evaluates to the value currently bound to that name (a variable,
a procedure/function value, a type, a module, a dynamic parameter, …). It is
never itself a call.

```
list-literal = '[' ( expression sep-by ',' ','? )? ']'
dict-literal = '{' ( dict-entry sep-by ',' ','? )? '}'
dict-entry   = ( IDENT | expression ) ':' expression
```

In a dict literal, a bare identifier key is shorthand for the corresponding
string key; a computed key is written as an expression followed by `:`.

### 6.2 Field access

```
field-access = expression '.' IDENT
```

`value.name` reads a field of a record, or a member of a module. It is **only**
field/member access — there is no method-call syntax in Doodle. Accessing a
non-existent field, or a field of a non-record/non-module, raises an error.

### 6.3 Indexing

```
index = expression '[' expression ']'
```

`x[k]` indexes a built-in sequence or mapping:

- **`List`** — integer index `k`, `0 <= k < length(list)`; O(1).
- **`Dict`** — the key `k` must be present; (amortized) O(1).
- **`String`** — integer index by **extended grapheme cluster** (§4.4),
  `0 <= k < length(s)`, yielding a length-one string (the k-th character a human
  perceives). Strings are variable-width, so this is O(k), not O(1); prefer
  iteration (`each`) to index loops for scanning.
- **`Bytes`** — integer index, yielding an `Int` in 0–255; O(1).

Out-of-range indices and absent keys raise. Indexing is built in for these types
only; it is not user-extensible in this version, and applying `[]` to any other
type raises. `[]` denotes indexed access, not necessarily constant-time access;
its cost is a property of the type.

### 6.4 Calls

```
call        = callee '(' arguments? ')' block-arg?
callee      = primary | field-access | index | call
arguments   = argument sep-by ','
argument    = expression                 # positional
            | IDENT ':' expression        # keyword
block-arg   = 'do' ( '(' block-params ')' )? body 'end'
block-params= IDENT sep-by ','
```

- Parentheses are **always required** to call, including zero-argument calls
  (`home()`). The callee is evaluated first, then the arguments left to right.
- **Positional** arguments bind to parameters by position; **keyword** arguments
  (`name: expr`) bind by parameter name. All positional arguments must precede
  all keyword arguments. Each parameter must receive exactly one argument
  (unless it has a default, §8.2). Supplying an unknown keyword, a duplicate
  argument, or the wrong arity raises an error.
- A trailing `do … end` supplies a **block argument** to a callee that declares
  a block parameter (§8.5). A zero-parameter block may omit the `()`.

Calling a **procedure** (a `to`) yields no value; using such a call where a value
is required is an error (§6.11). Calling a **function** (an `fn`) yields its
result.

Record construction (§9.2) uses call syntax with keyword arguments:
`Point(x: 3, y: 4)`.

### 6.5 Operators and precedence

Binary and unary operators bind according to the following table, from highest
precedence (binds tightest) to lowest. Associativity is noted.

| Level | Operators | Assoc. |
|---|---|---|
| 1 | postfix `.` `[]` `()` (access, index, call) | left |
| 2 | `**` (exponentiation) | right |
| 3 | unary `-`, unary `+` | — |
| 4 | `*` `/` `//` `%` | left |
| 5 | `+` `-` (binary) | left |
| 6 | `<` `>` `<=` `>=` `==` `!=` `is` | non-associative |
| 7 | `not` (unary) | — |
| 8 | `and` | left |
| 9 | `or` | left |

- `**` is right-associative: `2 ** 3 ** 2` is `2 ** (3 ** 2)` = `512`.
- Comparison operators are **non-associative**: `a < b < c` is a **static
  error**. Write `a < b and b < c`. (There is no chained comparison.)
- `and` and `or` **short-circuit** and require Boolean operands; the result is a
  Boolean. `not` requires a Boolean operand.
- `x is T` tests membership: `true` if `x`'s type is `T` (for a type value,
  nominal for records; `Number` matches `Int` or `Float`) or if `x`'s type
  implements the protocol `T` (for a protocol value); otherwise `false`.

There is **no operator overloading**. `+`, `-`, `*`, `/`, `//`, `%`, `**` apply
to numbers (with `+` and `*` additionally defined on strings for concatenation
and repetition, and `+` on lists/bytes as specified by the standard library
where applicable). Applying an arithmetic operator to operands for which it is
undefined (e.g. a string plus a number, or `<` on records) raises an error.

### 6.6 Boolean, comparison, and arithmetic semantics

- Equality/inequality (`==`, `!=`) are structural (§4.13) and total (defined for
  all pairs of values; unequal types compare unequal, never error).
- Ordering (`<`, `>`, `<=`, `>=`) is defined for numbers and strings; for
  strings it is code-point-lexicographic over the NFC form (§4.4) — not
  locale/dictionary collation, which is a standard-library concern. Applying
  ordering to values for which it is undefined raises.
- Logical `and`/`or`/`not` operate only on Booleans and short-circuit.

### 6.7 String interpolation

Within a string or triple-quoted string, `{ expression }` is replaced by the
textual rendering of the expression's value. The value is converted to a string
via the `to_string` hook (§15); built-in types and records have a rendering
supplied by the standard library. `{{` and `}}` denote literal braces. The
interpolated expression is any expression (it may contain further calls,
operators, and even strings) but **may not contain a line terminator**, in any
string form — a `{expr}` is confined to a single line. This keeps a missing `}`
from swallowing the rest of the file and, for triple-quoted strings, keeps
margin stripping (§3.6.4) from ever operating inside an expression; a long
expression binds a local first. An empty interpolation `{}` — or whitespace-only
`{ }` — is a static error whose diagnostic offers both intents: write an
expression, or `{{` for a literal brace. (Interpolating an empty dict is still
written `{ {} }`.)

A `#` **may not appear inside an interpolation** (in any string form). A `#`
comment runs to end of line (§3.3), which would swallow the closing `}`, so a
`#` inside `{…}` is a static error at the `#` — "a comment can't appear inside a
string's `{…}`" — suggesting the fix: move the comment outside the string, or
bind the expression to a named local. (Whitespace inside `{…}` is still
insignificant; only `#` is affected.)

### 6.8 `if` as an expression

`if` may be used as an expression (§7.5 gives the statement form). As an
expression it yields the value of the selected branch, and therefore every
branch — including a final `else` — must be present and must end in an
expression that produces a value:

```
let label = if n > 0 then "positive"
            else if n < 0 then "negative"
            else "zero"
            end
```

An `if` used in expression position without an `else`, or with a branch that
does not produce a value, is an error.

### 6.9 `try` as an expression

`try … rescue e … end` may likewise be used as an expression, yielding the value
of the body if it completes normally, or the value of the rescue body if an
error is caught (§12). In expression position both bodies must produce a value.

### 6.10 Anonymous functions

```
anon-fn = 'fn' '(' params? ')' body 'end'
```

An anonymous function is a first-class function value that closes over its
lexical environment. Its body follows the function rules (§8.4): the value of the
last expression is returned, and `return expr` returns early. Example:

```
let double = fn(x) x * 2 end
```

There is no anonymous *procedure* form; a block (§8.5) fills the role of
anonymous effectful code passed to a callee, and an anonymous function that
returns `nil` covers the rest.

### 6.11 Expression statements and the function/procedure distinction

An expression alone may stand as a statement (§7.2); its value is discarded.
Because a **procedure** call produces no value, a procedure call is a valid
statement but is an error in expression position (e.g. as the right-hand side of
`let`, an argument, or an operand). A **function** call is an expression.

A tool may warn when a non-final expression statement computes a value that is
then discarded (a likely mistake, such as writing `foo` where `foo()` was
meant); this is guidance, not a language error, since the interactive
environment legitimately evaluates bare expressions to display them.

---

## 7. Statements

### 7.1 Statement sequences

A body is a sequence of statements separated as in §3.2:

```
body = ( statement ( (NEWLINE | ';') statement )* )?
```

Within a **function** body (§8.4), the value of the last statement, if it is an
expression, is the function's result. Within a **procedure** body, statement
values are always discarded.

A statement is one of:

```
statement = let-stmt | const-stmt | assign-stmt
          | expression                     # expression statement
          | if-stmt | while-stmt | loop-stmt | with-stmt
          | try-stmt | return-stmt | break-stmt | continue-stmt | raise-stmt
          | to-decl | fn-decl
          | record-decl | protocol-decl | implement-decl
          | module-decl | parameter-decl
          | import-stmt | exports-stmt
```

Placement rules: `let`, `const`, `to`, and `fn` declarations may appear at module
level or nested within any body (a nested `to`/`fn` is lexically scoped to the
enclosing body). `record`, `protocol`, `implement`, `parameter`, `module`,
`import`, and `exports` appear at **module level only**.

### 7.2 Expression statement

An expression used as a statement evaluates the expression for its effect (and,
as the last statement of a function body, for its value).

### 7.3 `let` / `const`

See §5.2.

### 7.4 Assignment

See §5.3.

### 7.5 `if`

```
if-stmt = 'if' expression 'then' body
          ( 'else' 'if' expression 'then' body )*
          ( 'else' body )?
          'end'
```

The condition must be a `Bool`. `else if` is written as two words and chains
without additional `end`s: a whole `if`/`else if`/`else` chain is closed by a
single `end`. As a statement, `else` is optional; as an expression (§6.8), a
final `else` is required.

### 7.6 `while`

```
while-stmt = 'while' expression 'do' body 'end'
```

The condition (a `Bool`) is evaluated before each iteration; the body runs while
it is `true`. `while` is a keyword (rather than a library function) because its
condition must be evaluated repeatedly. There is no `until`, no `unless`, and no
`do`/`while`; the last is expressed with `loop` and `break` (§7.7).

### 7.7 `loop`

```
loop-stmt = 'loop' 'do' body 'end'
```

`loop do … end` repeats its body indefinitely; exit is via `break` (or `return`,
or a raised error). It expresses post- and mid-condition loops:

```
loop do
    let line = read_line()
    if line == "" then break end
    handle(line)
end
```

### 7.8 `with`

See §5.5.

### 7.9 `try` / `rescue` / `raise`

See §12.

### 7.10 `return` / `break` / `continue`

These are the three non-local exits, at three nesting levels (§8.5, §12):

```
return-stmt   = 'return' expression?
break-stmt    = 'break' expression?
continue-stmt = 'continue' expression?
```

- `return` exits the lexically enclosing procedure or function, unwinding
  through any intervening blocks. In a **function** it takes a value (`return
  expr`); in a **procedure** it takes none (`return`).
- `break` exits the nearest enclosing block-consuming call — a loop
  (`while`/`loop`), or a call that received the current block. An optional value
  becomes that call's value where applicable.
- `continue` ends the current block invocation. The block-consuming callee may
  invoke the block again (the next loop iteration, the next element, …). An
  optional value is the value this invocation yields to the callee (e.g. the
  mapped result in a mapping function).

Using `return` outside a procedure/function body, or `break`/`continue` outside
a block, is a static error. `with` bodies are single-invocation, so `continue`
and `break` within a `with` body both simply exit the body (undoing the dynamic
binding, §5.5).

---

## 8. Procedures and functions

Doodle distinguishes **procedures** (declared with `to`), which perform effects
and yield no value, from **functions** (declared with `fn`), which yield a value.
This distinction is load-bearing: it eliminates accidental return values and
makes intent visible at the declaration and every call site.

### 8.1 Declarations

```
to-decl = 'to' IDENT '(' params? ')' body 'end'
fn-decl = 'fn' IDENT '(' params? ')' body 'end'
```

`to name(…) … end` declares a procedure; `fn name(…) … end` declares a function.
Both introduce a callable value bound to `name` in the current scope.

### 8.2 Parameters

```
params      = param sep-by ','
param       = IDENT ( '=' expression )?     # ordinary (optionally defaulted)
            | 'do' IDENT                     # block parameter (at most one, last)
```

- An ordinary parameter may have a **default value** (`name = expr`), used when
  the argument is omitted. The default expression is evaluated at call time in
  the declaration's lexical scope.
- At the call site, an argument may bind to any ordinary parameter by position or
  by keyword (`name: value`, §6.4). This version does not distinguish
  positional-only from keyword-only parameters; every ordinary parameter may be
  supplied either way (Appendix D).
- A **block parameter** is written `do name` and, if present, must be the last
  parameter. It receives the trailing `do … end` block argument (§8.5) and is
  invoked as `name(args)` inside the body.

### 8.3 Invocation and argument binding

On a call (§6.4), arguments are bound to parameters: positional arguments fill
parameters left to right; keyword arguments fill by name; defaults supply any
omitted ordinary parameters; the block argument, if any, binds the block
parameter. A mismatch (missing required argument, unknown keyword, duplicate
binding, extra argument, block argument to a callee with no block parameter, or
missing block argument for a required block parameter) raises an error.

Argument binding follows the value/reference semantics of §4.14: a value-record
argument is copied into its parameter; a reference-typed argument is shared.

### 8.4 Return values and the fn/to distinction

- A **procedure** (`to`) yields no value. Its body is executed for effect; the
  value of its last statement is discarded. `return` (no value) exits early.
  Using a procedure call as an expression is an error (§6.11).
- A **function** (`fn`) yields a value: the value of the last expression
  evaluated in its body, or the operand of an executed `return expr`. Every path
  through a function body must produce a value; a function whose body can fall
  off the end without producing a value (e.g. ending in a `let` statement, or an
  `if` without `else` in tail position) is a static error where determinable and
  otherwise raises at runtime.

By convention, functions should avoid observable effects and procedures carry
them; the language does not enforce purity.

### 8.5 Blocks

A **block** is the `do … end` unit passed as the trailing argument of a call:

```
block-arg   = 'do' ( '(' block-params ')' )? body 'end'
```

Blocks are the primary control-abstraction mechanism: constructs that would be
keywords in other languages (iteration, resource scoping, retry, …) are ordinary
callables that take a block. A callee declares a block parameter (§8.2) and
invokes the block by name.

Blocks are **second-class**:

- A block may be passed only as the trailing block argument of a call. It is not
  a value: it cannot be bound with `let`, stored in a data structure, returned,
  or passed onward — except that it may be invoked in tail position, preserving
  tail calls (§8.7).
- A block **closes over variables** in its lexical environment (it can read and
  mutate them).
- The block's control exits are as in §7.10: `continue` ends the current block
  invocation (optionally yielding a value to the callee), `break` exits the
  block-consuming call, and `return` punches through the block to exit the
  enclosing procedure/function.
- Like a function, a block yields the value of its last expression on each
  invocation; whether that value is used is up to the callee (an iterating callee
  ignores it; a mapping callee collects it).

First-class "anonymous callable code" is provided instead by anonymous functions
(§6.10), whose `return` is local to the function and which may be stored and
passed freely.

Example — a user-defined block-consuming callee and its use:

```
to each(items, do body)
    let i = 0
    while i < length(items) do
        body(items[i])
        i = i + 1
    end
end

each(shapes) do(shape)
    draw(shape)
end
```

### 8.6 Docstrings

If the **first** element of a procedure body, function body, record body (§9),
or module/file body (§11) is a string literal (ordinary or triple-quoted), that
string is the unit's **docstring**. A docstring is not an executed statement; it
is attached as metadata and is available through reflection (§13). Docstrings
are the idiomatic use of triple-quoted strings.

```
fn distance(a, b)
    """Euclidean distance between two points."""
    let dx = b.x - a.x
    let dy = b.y - a.y
    sqrt(dx * dx + dy * dy)
end
```

### 8.7 Proper tail calls

A conforming implementation **must** implement proper tail calls: a call in tail
position executes in constant control-stack space relative to the caller. This
makes recursion a sound primary means of iteration.

A call is in **tail position** when its result is the result of the enclosing
function with no further computation pending, specifically:

- the last expression of a function body (or of a block invocation, with respect
  to that invocation);
- the selected branch of an `if` that is itself in tail position;
- the body position handed to a block-consuming callee, with respect to that
  block's invocation.

A call is **not** in tail position when work remains after it returns, including:

- an operand of an operator or an argument of another call (`f(x) + 1`, `g(f(x))`);
- the right-hand side of a `let`/assignment whose binding is then used;
- any call inside a `with` body (the dynamic binding must be restored on exit,
  §5.5) or inside a `try` body (a handler must be unwound), which therefore does
  **not** preserve the tail-call property for calls within them.

A conforming implementation should keep a bounded history of elided tail frames
so that diagnostics for later errors remain useful (informative).

---

## 9. Records

Records are named product types: pure data with named, mutable fields and no
attached behavior (behavior is expressed as procedures/functions that take the
record, §1.3, and, where polymorphism is needed, as protocols, §10).

### 9.1 Declaration

```
record-decl = 'ref'? 'record' IDENT 'with' field-list body? 'end'
field-list  = IDENT sep-by ','
```

`record Name with f1, f2, … end` declares a **value-typed** record; prefixing
`ref` declares a **reference-typed** record (§4.14). The optional trailing
`body` may contain only a leading docstring in this version (no methods; see
Appendix D). The declaration introduces a type/constructor value named `Name`
(§4.12) at module level.

```
record Point with x, y end
ref record Turtle with position, heading, pen_down end
```

### 9.2 Construction

A record is constructed by calling its type value with **keyword arguments**
naming its fields:

```
let p = Point(x: 3, y: 4)
```

Every declared field must be supplied exactly once; supplying an unknown field,
omitting a field, or duplicating one raises an error. Fields are constructed in
declaration order.

> Whether fields may have declaration-site defaults, and whether positional
> construction is allowed, is deferred; this version requires all fields as
> keyword arguments (Appendix D).

### 9.3 Field access and mutation

`r.field` reads a field (§6.2); `r.field = value` mutates it (§7.4). Reading or
writing an undeclared field raises. Field mutation is available for both value
and reference records; the difference between them is only whether the record
was copied or shared on binding (§4.14).

### 9.4 Equality and identity

Records use the language's structural `==` (§4.13): equal only to a record of the
**same type** with pairwise-equal fields. This applies to reference records too;
reference identity is a separate question tested by a standard-library function
(`is_same`).

### 9.5 Default protocol behavior

By default every record participates in the well-known protocols the standard
library defines for rendering and hashing (§15): a record has a default textual
rendering (used by interpolation and display) and is usable as a dict key. A
record type may additionally *implement* protocols (e.g. to become iterable) as
in §10; it does not do so by default.

---

## 10. Protocols

A **protocol** is a named set of procedure/function signatures that a type may
implement, providing the language's single mechanism for polymorphism over
types. Protocols let user-defined types participate in operations on the same
footing as built-in types, without classes, inheritance, methods-on-data, or
operator overloading.

### 10.1 Declaration

```
protocol-decl = 'protocol' IDENT ( 'extends' IDENT )? proto-body 'end'
proto-body    = ( docstring )? proto-member*
proto-member  = ( 'to' | 'fn' ) IDENT '(' params ')' ( body )? 'end'?
```

A protocol lists members as `to`/`fn` signatures. A member **without** a body is
**required**: an implementing type must provide it. A member **with** a body is a
**default**: implementations inherit it but may override it. The `to`/`fn` choice
declares whether the member is effectful (no result) or value-producing, exactly
as for ordinary callables.

By convention the first parameter is named `self` and is the value the call
dispatches on; `self` is not a keyword and any name may be used.

`protocol Child extends Parent` declares that implementing `Child` requires also
implementing `Parent` (a requirement relationship, not code inheritance). Default
members of `Parent` are available to `Child` implementors through the ordinary
default-member mechanism.

```
protocol Iterable
    """Types whose elements can be visited in order."""
    to each(self, do body)          # required
end
```

### 10.2 Implementation

```
implement-decl = 'implement' IDENT 'for' IDENT impl-body 'end'
impl-body      = ( to-decl | fn-decl )*
```

`implement P for T … end` provides `T`'s implementations of `P`'s required
members (and any overrides of defaults). It is a module-level declaration and
registers the `(type, protocol member) → callable` associations at load time.

```
implement Iterable for Range
    to each(r, do body)
        let i = r.start
        while i < r.stop do
            body(i)
            i = i + r.step
        end
    end
end
```

A type must implement all of a protocol's required members (and those of any
protocol it `extends`); a partial or missing implementation is an error.

### 10.3 Dispatch

Protocol members are called like ordinary callables, dispatching on the
**runtime type of the first argument** (single dispatch):

```
each(my_range) do(i) … end     # dispatches on my_range's type
```

- If the first argument's type does not implement the member's protocol, the call
  raises an error naming the missing implementation.
- If the same member name is provided by two protocols both implemented by the
  argument's type (ambiguity), the unqualified call is an error; the caller
  disambiguates with a qualified form `Protocol.member(args)`.

The qualified form `P.member(args)` selects protocol `P` explicitly and is always
available.

### 10.4 Built-in / well-known protocols

The language mechanism above is complete on its own. A small number of *well-known
protocols* are relied upon by core operations (string interpolation and value
display; dict-key hashing) and are defined, together with their default
implementations for built-in types, by the **standard library**, not here. Their
required method names are fixed by §15 so the language can refer to them. This
version anticipates the well-known protocols `Stringable`, `Hashable`,
`Iterable`, and `Lengthable`; their full definitions belong to the standard
library specification.

---

## 11. Modules and imports

### 11.1 Modules and packages

Each source file is a **module**; its name is the file's base name (which must be
a valid module name, §3.4). A directory of modules is a **package**; nested
directories give nested packages, so a dotted path such as `geometry.shapes`
names `geometry/shapes.doodle`. The file/directory structure is the
namespace structure.

A module's members are its module-level definitions. A module may begin with a
docstring (§8.6). By default all module-level definitions are public. A module
may restrict its public surface with an `exports` declaration:

```
exports-stmt = 'exports' IDENT sep-by ','
```

listing the names other modules may see; names not listed are private to the
module.

An explicit `module Name … end` block form is also provided:

```
module-decl = 'module' IDENT body 'end'
```

It may be used to wrap a file's contents under an explicit name (in which case
`Name` should equal the file's base name) or to declare a nested module. The
implicit "file is a module" model is primary; the explicit form is available for
documentation and nesting. (The precise interaction of file-level and block-level
modules is provisional; see Appendix D.)

### 11.2 Import forms

```
import-stmt = 'import' import-target sep-by ','
import-target
            = module-path ( 'as' IDENT )?          # qualified (optionally renamed)
            | module-path '.' IDENT ( 'as' IDENT )? # single member (optionally renamed)
            | module-path '.*'                       # all exported members
module-path = IDENT ( '.' IDENT )*
```

There is one import keyword, with three target forms and comma separation:

- `import shapes` binds the module value to `shapes`; access members as
  `shapes.circle`.
- `import shapes as s` binds the module value to `s`.
- `import shapes.circle` binds the member `circle` into the current scope; `import
  shapes.circle as c` renames it.
- `import shapes.*` binds all of `shapes`'s exported members into the current
  scope under their own names. `.*` may not be renamed with `as`.
- Comma-separated targets are equivalent to separate `import` statements:
  `import shapes.circle, shapes.square as sq, colors`.

The dotted form in an import mirrors member access: `import m.x` brings in the
same thing you would otherwise reach as `m.x`. Wildcard imports (`.*`) are the
idiomatic way to bring a domain vocabulary into scope (an editor's default
template typically begins with such imports); selective imports are for occasional
needs and name-collision resolution.

### 11.3 Loading semantics

Import forms concern **name binding**; **loading** is separate and happens once
per module:

- The first time any import references a module, the module is *loaded*: its
  top-level code runs, in order, for effect (defining its members and any
  module-level state). Subsequent references bind names against the
  already-loaded module without re-running it (**singleton** semantics). All
  importers share one module instance and its state.
- Because loading runs code, modules **should** be quiet at load time — define
  procedures/functions/records, declare parameters with defaults, and set up
  members — and defer observable effects to explicitly invoked entry points.
- Imports do **not** cascade: `import a` binds only `a` (or its selected
  members); it does not re-export names that `a` itself imported.
- **Circular imports are an error.** If loading module `a` requires loading `b`,
  and loading `b` requires `a` before `a` has finished loading, loading fails
  with a diagnostic. Refactor the shared portion into a third module.
- A module may hold mutable module-level state (via `let`); `const` declares
  module-level constants. Such state is shared across all importers (a
  consequence of singleton loading).

Reloading an already-loaded module (e.g. after an edit, in an interactive
session) is an environment feature, not a language operation, and is out of
scope here.

### 11.4 The prelude

Doodle does not build any names into the language: identifiers such as `print`,
`each`, and `length` are ordinary standard-library names. The set of names
implicitly available in every program (the *prelude*), and any additional
per-project or editor-provided preambles, are defined by the standard library and
tooling — not by this specification.

---

## 12. Errors and exceptions

Doodle's error facility is deliberately minimal. The intended discipline is that
*expected* failures are communicated by ordinary return values (e.g. `nil` or a
result record), and the exception mechanism is reserved for genuinely exceptional
conditions — bugs and unrecoverable situations — mostly handled, if at all, at
the top of a program.

### 12.1 Raising

```
raise-stmt = 'raise' expression?
```

`raise value` raises `value` as an exception, aborting the current computation
and unwinding the call stack until a handler is found or the program terminates.
The value is typically a record (carrying structured information) or a string.
A bare `raise` is valid **only** inside a rescue body, where it re-raises the
exception currently being handled, preserving its original stack trace.

The stack trace is captured at the point of `raise` (not lazily), so it reflects
where the error originated.

### 12.2 Handling

```
try-stmt = 'try' body 'rescue' IDENT body 'end'
```

`try B rescue e H end` runs `B`; if `B` raises, the raised value is bound to `e`
and the handler `H` runs. There is a single, untyped rescue clause: to handle
particular kinds of error, inspect `e` in the handler (e.g. `if e is SomeError
then … else raise end`), which composes with `is` (§6.5) and re-raise. `try` may
be used as an expression (§6.9).

There is no `ensure`/`finally` clause in the language. Cleanup that must run on
every exit is expressed with a block-consuming helper (a `with`-style callable)
that performs its cleanup as its block returns or unwinds; such helpers are
standard-library facilities built on the block-call mechanism, which guarantees
cleanup on normal completion, exception, and non-local exit (§5.5, §8.5).

### 12.3 Propagation and uncaught errors

An exception propagates outward through calls, blocks, `while`/`loop`/`with`
bodies, and function/procedure boundaries until a `try` handles it. `with`
bindings are undone as their bodies unwind (§5.5). `return`, `break`, and
`continue` are non-local *exits*, not exceptions, and are not caught by `rescue`.

An uncaught exception terminates the program (or, interactively, the current
evaluation) with a diagnostic that includes the raised value and the captured
stack trace.

---

## 13. Reflection

Procedures, functions, record types, modules, and protocols are values (§4) and
support a bounded amount of runtime introspection. The concrete accessor
functions belong to the standard library; the language guarantees only that the
following information is available for reflection and diagnostics:

- **Callables:** name (empty for anonymous functions); whether it is a procedure
  or a function; its parameters (count and names, defaults, and whether it takes
  a block); source location (file and line of definition); docstring, if any.
- **Record types:** name; field names in declaration order; whether value- or
  reference-typed; source location; docstring.
- **Modules:** name; source location; the set of exported names and their values;
  docstring.
- **Protocols:** name; member signatures; source location; docstring.
- **Any value:** its type, via the `is` operator (§6.5) and a standard-library
  `type_of`.

The **source text** or abstract syntax of a callable is deliberately **not**
available: reflection exposes structure and provenance, not code. Modules cannot
be mutated through reflection (their export set is fixed; §4.11); there is no
monkey-patching.

---

## 14. Execution model

A program is executed by loading and running a designated entry module (§11.3).
Evaluation is **single-threaded** and deterministic with respect to evaluation
order: within an expression, operands and arguments are evaluated left to right;
statements run in sequence; `and`/`or` short-circuit (§6.5). This version of the
language has no concurrency, parallelism, or asynchrony.

The language core has no built-in input/output, graphics, timing, or operating
system access. Such capabilities are provided to a program by its host — the
standard library and the embedding environment — through ordinary
procedures/functions and modules. The separation of the pure language core from
the host environment (including the interactive canvas/REPL and the embedding
API) is an explicit design goal but is specified elsewhere.

---

## 15. Interface with the standard library (normative hooks)

Although the standard library is out of scope, a few language constructs are
defined in terms of it. This section fixes those coupling points so the two
specifications agree. For each hook, the language guarantees the *call*; the
standard library must supply the *behavior*.

1. **Value rendering (`to_string`).** String interpolation (§6.7) and any
   language-driven textual display of a value convert that value to a `String` by
   invoking the well-known `Stringable` protocol's `to_string` member on it. The
   standard library must define `Stringable`, implement it for all built-in types,
   and provide a default implementation for records.

2. **Hashing and dict keys (`hash`).** The built-in `Dict` type (§4.8) requires
   its keys to be hashable. Hashability is expressed by the well-known `Hashable`
   protocol's `hash` member together with the language's structural equality
   (§4.13). The standard library must define `Hashable`, implement it for the
   hashable built-in types, and provide a record default. Using a
   non-hashable value (e.g. a list) as a dict key raises.

3. **String representation and Unicode data.** The language owns Unicode data —
   NFC normalization, extended-grapheme-cluster segmentation (UAX #29), and
   case/property tables — as irreducible runtime primitives, in the same sense as
   arbitrary-precision arithmetic and I/O. It exposes their *results*, so the
   standard library and user code build on documented capabilities rather than
   hidden ones:
   - **NFC** is applied whenever a `String` value is constructed (§4.4).
   - **Grapheme access** is the default: `length`, `s[i]`, and iteration over a
     `String` operate on grapheme clusters, so the standard library's
     `Lengthable` and `Iterable` implementations for `String` rest on this
     primitive.
   - **The byte view** is the one irreducible representation primitive: the
     language converts between a `String` and its NFC UTF-8 `Bytes`, the decoding
     direction validating UTF-8 and normalizing (§4.4, §4.5). It round-trips —
     encoding a string and decoding the result yields the same string.
   - **Code points** need no further primitive: UTF-8 decoding is table-free, so
     the standard library derives a string's code points from its byte view in
     ordinary Doodle — a demonstration that the byte/code-point layering is not
     magic.

   Core syntax does not itself iterate (there is no `for` loop); the `Iterable`
   and `Lengthable` protocols and the functions built on them (`each`, `map`,
   `length`, …) are standard-library, named here only so the two specifications
   share a vocabulary.

4. **Prelude and core functions.** Every named value a program uses — including
   `print`, `read_line`, `assert`, `type_of`, `is_same`, `copy`, and all
   collection and math operations — is provided by the standard library and the
   prelude (§11.4), not by the language.

Any additional well-known protocol or hook must be introduced by amending this
section, so that the language↔library boundary stays explicit.

---

## Appendix A. Grammar summary

Lexical nonterminals (`INT`, `FLOAT`, `STRING`, `TRIPLE_STRING`, `BYTES`,
`IDENT`, etc.) are defined in §3. `NEWLINE` is a statement terminator (§3.2).

```
program     = ( module-item ( sep module-item )* )?
sep         = NEWLINE | ';'

module-item = statement            # subject to placement rules in §7.1

body        = ( statement ( sep statement )* )?

statement   = let-stmt | const-stmt | assign-stmt
            | expression
            | if-stmt | while-stmt | loop-stmt | with-stmt
            | try-stmt | return-stmt | break-stmt | continue-stmt | raise-stmt
            | to-decl | fn-decl
            | record-decl | protocol-decl | implement-decl
            | module-decl | parameter-decl
            | import-stmt | exports-stmt

let-stmt    = 'let' IDENT '=' expression
const-stmt  = 'const' IDENT '=' expression
assign-stmt = lvalue '=' expression
lvalue      = IDENT | expression '.' IDENT | expression '[' expression ']'

if-stmt     = 'if' expression 'then' body
              ( 'else' 'if' expression 'then' body )*
              ( 'else' body )? 'end'
while-stmt  = 'while' expression 'do' body 'end'
loop-stmt   = 'loop' 'do' body 'end'
with-stmt   = 'with' IDENT '=' expression 'do' body 'end'
try-stmt    = 'try' body 'rescue' IDENT body 'end'
return-stmt = 'return' expression?
break-stmt  = 'break' expression?
continue-stmt = 'continue' expression?
raise-stmt  = 'raise' expression?

to-decl     = 'to' IDENT '(' params? ')' body 'end'
fn-decl     = 'fn' IDENT '(' params? ')' body 'end'
anon-fn     = 'fn' '(' params? ')' body 'end'
params      = param ( ',' param )*
param       = IDENT ( '=' expression )? | 'do' IDENT

record-decl = 'ref'? 'record' IDENT 'with' IDENT ( ',' IDENT )* body? 'end'
protocol-decl = 'protocol' IDENT ( 'extends' IDENT )? proto-body 'end'
proto-body  = docstring? proto-member*
proto-member= ( 'to' | 'fn' ) IDENT '(' params? ')' body? 'end'?
implement-decl = 'implement' IDENT 'for' IDENT ( to-decl | fn-decl )* 'end'
module-decl = 'module' IDENT body 'end'
parameter-decl = 'parameter' IDENT '=' expression
exports-stmt = 'exports' IDENT ( ',' IDENT )*

import-stmt = 'import' import-target ( ',' import-target )*
import-target = module-path ( 'as' IDENT )?
              | module-path '.' IDENT ( 'as' IDENT )?
              | module-path '.*'
module-path = IDENT ( '.' IDENT )*

expression  = or-expr
or-expr     = and-expr ( 'or' and-expr )*
and-expr    = not-expr ( 'and' not-expr )*
not-expr    = 'not' not-expr | cmp-expr
cmp-expr    = add-expr ( cmp-op add-expr )?          # non-associative
cmp-op      = '==' | '!=' | '<' | '>' | '<=' | '>=' | 'is'
add-expr    = mul-expr ( ('+' | '-') mul-expr )*
mul-expr    = unary-expr ( ('*' | '/' | '//' | '%') unary-expr )*
unary-expr  = ('-' | '+') unary-expr | pow-expr
pow-expr    = postfix-expr ( '**' unary-expr )?       # right-assoc
postfix-expr= primary ( '.' IDENT
                      | '[' expression ']'
                      | '(' arguments? ')' block-arg? )*
primary     = INT | FLOAT | STRING | TRIPLE_STRING | BYTES
            | 'true' | 'false' | 'nil'
            | IDENT
            | '[' ( expression ( ',' expression )* ','? )? ']'
            | '{' ( dict-entry ( ',' dict-entry )* ','? )? '}'
            | '(' expression ')'
            | anon-fn
            | if-expr | try-expr
dict-entry  = ( IDENT | expression ) ':' expression
if-expr     = 'if' expression 'then' body
              ( 'else' 'if' expression 'then' body )*
              'else' body 'end'
try-expr    = 'try' body 'rescue' IDENT body 'end'
arguments   = argument ( ',' argument )*
argument    = expression | IDENT ':' expression
block-arg   = 'do' ( '(' IDENT ( ',' IDENT )* ')' )? body 'end'
docstring   = STRING | TRIPLE_STRING
```

This summary is a guide; §3–§13 are authoritative where they differ.

---

## Appendix B. Reserved words

The 35 keywords carried over from the design discussion:

```
and  as  break  const  continue  do  else  end  exports  extends
false  fn  for  if  implement  import  is  let  loop  module
nil  not  or  protocol  raise  record  ref  rescue  return  then
to  true  try  while  with
```

Added by this specification:

```
parameter          # dynamic-parameter declaration (§5.5)
```

Reserved but unused in this version (held for possible future use; §3.5):

```
next  use
```

`self` is **not** reserved; it is a naming convention for the dispatch argument
of protocol members (§10.1).

---

## Appendix C. Operator precedence (summary)

Highest (binds tightest) to lowest:

1. postfix `.`  `[]`  `()`
2. `**` (right-associative)
3. unary `-`  `+`
4. `*`  `/`  `//`  `%`
5. `+`  `-`
6. `==`  `!=`  `<`  `>`  `<=`  `>=`  `is` (non-associative)
7. `not`
8. `and`
9. `or`

Assignment `=` is a statement, not an operator. There is no ternary operator and
no operator overloading.

---

## Appendix D. Specification decisions and open issues

This appendix records choices this specification made to fill gaps in the design
discussion, and the questions that remain genuinely open. Items here are the most
likely to change.

### D.1 Decisions made here (not fully settled in the discussion)

- **`parameter` keyword (§5.5).** The discussion settled on lexical scope by
  default with dynamic parameters via `with`, sigil-free reads, and
  pre-declaration with defaults (its "Option B"), but never added a declaration
  keyword to the enumerated list. Because sigil-free reads require the compiler to
  know a name is dynamic, this specification introduces `parameter name =
  default` and adds `parameter` to the reserved words (bringing the count to 36).
- **`/` always yields `Float` (§4.2).** The discussion said `/` "produces a float
  when the result isn't a whole number" but pointed at Python 3, where `/` is
  always float. This spec adopts the Python-3 rule (`4 / 2` is `2.0`) as the
  simpler, more predictable one.
- **Division/modulo by zero raises (§4.2).** Consistent with the "loud failure"
  stance and the discussion's listing of divide-by-zero among raising errors.
- **`next` and `use` reserved-but-unused (§3.5, Appendix B).** The discussion
  chose `continue` over `next` and removed `open`/`from`/`use` from imports.
  Rather than free these words immediately, this spec reserves `next` and `use`
  to avoid churn if they return; `open` and `from` are **not** reserved. This
  mildly contradicts the discussion's "nothing reserved for future use" note and
  can be reversed.
- **Module block vs. file module (§11.1).** The discussion asserted "one file =
  one module" and also showed a file wholly wrapped in `module Name … end`. This
  spec makes the implicit file-module primary and keeps `module … end` for
  explicit naming/nesting, with the constraint that a file-wrapping `module`
  name match the file. The precise semantics of nested modules are thin and may
  need revisiting.
- **Parameter/argument model (§8.2, §8.3).** The discussion adopted keyword
  arguments and mentioned optional parameters but never fixed the declaration
  syntax or the positional/keyword distinction. This spec: ordinary parameters
  may be supplied positionally or by keyword; defaults are written `name = expr`
  in the declaration; there is no positional-only/keyword-only distinction yet.
- **Record construction is keyword-only; no field defaults (§9.2).** Chosen for
  clarity; positional construction and field defaults are candidate extensions.
- **Value-record copy is recursive over value fields, sharing references
  (§4.14).** The discussion said "shallow copy" without pinning nesting; this
  spec gives the C-struct-like rule.
- **Built-in type value spellings (§4.12).** `Int`, `Float`, `Number`, `String`,
  `Bytes`, `Bool`, `Nil`, `List`, `Dict`, `Procedure` are provisional names.
- **String model: grapheme-default, immutable, NFC (§4.4, §6.3, §4.13, §15).**
  The discussion deferred this. Resolved: strings are immutable and stored in
  NFC; the default character is the extended grapheme cluster, so `length`,
  `s[i]`, and iteration are grapheme-based (`s[i]` is human-correct but O(i));
  `==` is canonical (NFC) equivalence; `<` is code-point-lexicographic over the
  NFC form. Consequence, adopted deliberately: `String` is a *normalized
  human-text* type and is lossy with respect to exact bytes/code points, while
  `Bytes` is the faithful container. The language exposes a String↔UTF-8 `Bytes`
  primitive as the no-magic floor; code-point access is derived from it in the
  standard library.
- **Dict iteration order = insertion order (§4.8).** Left unspecified by the
  discussion. Fixed to insertion order — the modern default (Python 3.7+, Ruby),
  intuitive and stable — and required to be deterministic so that engine-level
  replay and time-travel work (engine specification §11).
- **Source-position units (§3.1; resolves implementation-plan Appendix C S-1).**
  The discussion left source positions unspecified. Positions are defined over
  the NFC-normalized source and measured in Unicode code points — a 1-based line
  and a 1-based code-point column; spans are code-point ranges. This is the unit
  L and the engine (E§8.1) report; a binding or host may convert to bytes,
  UTF-16 code units, or grapheme clusters.
- **Identifiers use `XID` not `ID` (§3.4).** The discussion (and an earlier
  draft) wrote `ID_Start`/`ID_Continue`; fixed to `XID_Start`/`XID_Continue`
  (UAX #31's recommended default). The `XID` properties are closed under NFC,
  which the plain `ID` properties are not — necessary because source is
  NFC-normalized (§3.1) and identifiers are compared by NFC equality (§3.4).
- **Line endings canonicalized to LF on load (§3.1).** Left unstated. A CRLF is
  replaced by a single LF before tokenization, so positions/spans and the lexer
  see LF-only text; the on-disk file may use either LF or CRLF (§3.2).
- **Continuation-trigger token set (§3.2; resolves implementation-plan Appendix
  C S-2).** The draft §3.2 said a line continues when it "ends with a binary
  operator" without enumerating the set. Resolved: the triggers are exactly the
  binary arithmetic/comparison operators (`+ - * / // % ** == != < > <= >=`),
  the comma, and the word operators `and`/`or`/`is`. A trailing `-`/`+`
  continues (continuation keys on the token, not on its unary/binary role, which
  is a parse-time distinction); the assignment `=`, the unary `not`, the access
  `.`, and the punctuation `:` do **not** continue. A trailing comment between
  the operator and the newline is transparent. Only the trailing token matters;
  a leading operator never joins to the previous line.
- **Numeric-literal lexing (§3.6.1/§3.6.2).** The EBNF `(DIGIT | '_')*` formally
  permitted `1_`/`1__0`/`0x_F`, contradicting the prose. Resolved
  prose-authoritative: an underscore must fall between two digits. A base prefix
  requires a valid digit (`0x`, `0xG` are static errors); prefixes are lowercase
  (`0XFF` is not a literal); an exponent requires a digit (`1e`, `1e+` are
  errors); a base-10 literal may have leading zeros (no C-style octal).
- **String escapes, interpolation newlines, and empty `{}`
  (§3.6.3/§3.6.4/§3.6.5/§6.7; resolves implementation-plan Appendix C
  S-47/S-48/S-49).** Left underspecified by the draft. Resolved: (1) the escape
  set is **closed** — an unknown escape (`\q`) is a static error naming it, and a
  malformed known escape (`\x` short, or braceless/empty/over-long `\u`) is a
  distinct one; `\xHH` denotes code point `U+00HH` in a string and byte `0xHH`
  in `b"…"` (Rust's ≤`0x7F` restriction on string `\x` was considered and
  rejected — the String/Bytes split already prevents the confusion it guards).
  (2) An interpolation `{expr}` may not contain a line terminator in **any**
  string form — one uniform rule that preserves end-of-line error recovery and
  keeps §3.6.4 margin stripping out of expression interiors. (3) An empty
  `{}`/`{ }` is a static error offering both intents; `{ {} }` still interpolates
  an empty dict.

### D.2 Genuinely open (deferred by the discussion)

- **String collation and the full string API.** Core `<` on strings is a simple
  code-point ordering over the NFC form, deliberately *not* locale/dictionary
  collation; locale-aware collation, case mapping, splitting, searching, and the
  rest of the string API are standard-library (resting on the runtime's Unicode
  data, §15). The specific Unicode version the runtime targets is pinned per
  release; grapheme counts can therefore shift across Unicode versions.
- **The standard library.** The prelude, the four well-known protocols and their
  defaults, all collection/math/turtle facilities, `assert`, `copy`, `is_same`,
  `type_of`, byte buffers, fixed-width integers, and bitwise operations — the
  entire adjacent specification.
- **Interactive environment and host interface.** REPL, canvas, animation,
  time-travel debugging, execution visualization, module reloading, and the
  embedding/FFI API. These are explicitly separate from the language core.
- **Concurrency.** Excluded from this version by design; if added later it is a
  separate design effort.
- **Growth-path features.** Scripting/headless mode, file/OS/process access, and
  any performance-oriented facilities are future considerations that may
  influence later language revisions.
- **`ref record` and boxing.** The discussion noted that, in a dynamically typed
  language, a single generic reference cell suffices in principle; `ref record`
  is kept for the intent it communicates at the declaration site. Whether to also
  expose a generic `Box` remains open.
- **Re-export.** No syntax exists yet for a module to re-export a name it
  imported; deferred.
```
