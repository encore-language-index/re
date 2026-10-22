# re

Compiled regular expressions written in safe Encore. The package only uses
`core`; it has no native bindings, runtime file access, or external dependencies.

```encore
import core::result::Result
import re::Regex
import std::io::println

fn main() -> u32 {
    match Regex::new("[а-я]+") {
        Result::Ok(regex) => {
            for found in regex.find_iter("Привет, мир!") {
                println(found.as_str())
            }
        }
        Result::Err(error) => {
            println(error.message())
            ret 1_u32
        }
    }
    ret 0_u32
}
```

## API

`Regex::new(pattern)` returns `Result[Regex, RegexError]`. Compile once and reuse
the result. `Regex::with_options(pattern, options)` accepts flags built with:

```encore
let options = RegexOptions::new()
    .with_ignore_case(true)
    .with_multi_line(true)
    .with_dot_matches_new_line(true)
```

Import `RegexOptions` from `re`. All flags default to `false`.

| Operation | Result |
| --- | --- |
| `regex.is_match(text)` | `bool`: whether any match exists |
| `regex.find(text)` | `Option[Match]`: the first match |
| `regex.full_match(text)` | `bool`: whether the entire input matches |
| `regex.find_iter(text)` | Lazy, non-overlapping `FindIter` |
| `regex.matcher()` | `Matcher<H>` with reusable scratch buffers |

`Match.start()` and `.end()` are **byte offsets**, with an exclusive end;
`.as_str()` returns the matched substring. Matches retain their input and stay
valid after the matcher is used again. Substring extraction is deferred until
`as_str()` is called.

A matcher provides `is_match`, `find`, and `full_match`, using `mut Self&`:

```encore
let mut matcher = regex.matcher()
let first = matcher.find("first input")
let second = matcher.find("second input")
```

`matcher.find_from(text, byte_offset)` searches from a scalar boundary, preserving
the original input's anchor and word-boundary context. It returns `None` for an
out-of-range offset or an offset in a UTF-8 continuation byte.

Compiled regex copies share immutable program storage. Matchers own mutable
scratch buffers: use a separate matcher per concurrent search. Each `find_iter`
call creates its own workspace. `FindIter.next()` follows Encore's standard
`(next_iterator, Option[Match])` protocol and implements `Iterator`/`IntoIterator`.

## Syntax and matching rules

| Syntax | Meaning |
| --- | --- |
| `привет`, `🙂` | UTF-8 literals, matched exactly |
| `.` | One Unicode scalar; excludes LF unless dot-matches-new-line is enabled |
| `[a-zа-я]`, `[^abc]` | Union of literals/ranges, or its complement |
| `a\|b` | Alternative; the left branch has priority |
| `(...)`, `(?:...)` | Structural groups, without capture extraction |
| `*`, `+`, `?`, `{n}`, `{n,}`, `{n,m}` | Greedy repetition |
| `*?`, `+?`, `??`, `{n,m}?` | Lazy repetition; also supported for other counted forms |
| `^`, `$` | Input start/end; line start/end with multi-line enabled |
| `\A`, `\z` | Absolute input start/end |
| `\b`, `\B` | ASCII word boundary, or its inverse |
| `\d`, `\D` | ASCII digits `[0-9]`, or their complement |
| `\w`, `\W` | ASCII `[A-Za-z0-9_]`, or its complement |
| `\s`, `\S` | ASCII HT, LF, VT, FF, CR and space, or their complement |
| `\n`, `\r`, `\t`, `\f`, `\v`, `\a`, `\0` | Control characters, including NUL |
| `\xHH`, `\u{H…}` | A scalar encoded as two hex digits, or one to six hex digits |

In Encore source, double regex backslashes: `"\\d+"`, `"\\bword\\b"`.
An actual NUL in an Encore string is written `"\x00"`.
Punctuation may be escaped. Unknown escapes are errors; `\b` inside a character
class means backspace. Use `\]` for a class's literal closing bracket and put
`-` first/last or escape it. Empty classes are errors.

The engine chooses the earliest start, then follows alternative and repetition
priority. `a|ab` finds `a` in `ab`; `ab|a` finds `ab`. `full_match` considers all
branches while requiring both input boundaries, so both expressions fully match
`ab`. `$` is strict at the input end by default: `a$` does not match `"a\n"`.

UTF-8 scalars, not grapheme clusters, are the matching unit. ASCII ignore-case
equates `A` with `a`, including in ranges and negated classes; it does not equate
`А` with `а`, or `ß` with `ss`. No normalization is performed. Patterns and inputs
are expected to contain valid UTF-8; searches do not add a separate validation
pass. Valid UTF-8 input always produces spans on scalar boundaries.

Empty patterns, empty groups and empty alternatives are supported. After an
empty iterator match, the next search advances one entire scalar. An empty match
at EOF is yielded once. For example, `a*` on `a🙂` yields `0..1`, `1..1`, `5..5`.

Backreferences, lookaround, named groups, capture extraction, inline flags,
possessive quantifiers, Unicode properties, replacement and splitting are not
part of this version. Unsupported syntax produces a compile error instead of
silently selecting a different meaning. Character-class intersection and
subtraction operators are not supported; class punctuation denotes literals.

## Complexity and limits

Patterns compile to an ordered Thompson NFA. Search maintains deduplicated
state lists and uses an explicit stack for epsilon transitions. There is no
backtracking recursion and no per-character allocation after workspace creation.
Pattern compilation uses a bounded recursive parser for groups and an iterative
compiler for expression trees and expanded repeats.

A single search takes at most `O(m × n)` time and `O(m)` scratch space, where
`m` is expanded program size and `n` is input byte length. Pure literals use
linear KMP search. Required literal prefixes and a conservative set of starting
characters skip impossible candidates. ASCII classes use four 32-bit masks;
other scalars use sorted, merged ranges.

Repeated greedy searches made by `find_iter` can take `O(m × n²)` in total; the
single-search bound is not a claim that all iteration is linear. Match text is
not copied during scanning.

Compilation rejects patterns above 64 KiB, group nesting above 128, repetition
counts above 65,536 and expanded compilation cost above 65,536. The cost includes
structural/empty nodes as well as instructions, so rejection can occur before
65,536 actual instructions. These limits also bound work for nested empty
repetitions. `RegexError.message()` explains the failure and `.position()` gives
its byte offset in the pattern. Matching has no time limit.

## Local measurements

Measured on AMD Ryzen 7 7700, Linux x86-64, Encore 0.1.6-neumann, `extreme`
profile. Timings are local observations, not performance guarantees. Inputs were
built before timing. Compilation means are over 100 compilations; search means
are over the iteration counts shown. Initial searches include scratch creation;
reused searches use one `Matcher`. All search timings measure `is_match`, which
can stop at the first accepting state. Timings exclude process startup and compilation
of the Encore program itself.

| Pattern / input | Compile mean | First search | Reused search mean | Iterations |
| --- | ---: | ---: | ---: | ---: |
| `needle`, 1 MiB of `a` followed by `needle` | 8.35 µs | 10.32 µs | 7.65 µs | 5 |
| `[0-9]+`, 1 MiB of `a`, no match | 2.58 µs | 24.37 ms | 24.16 ms | 3 |
| `(a+)+b`, 16 KiB of `a`, no match | 4.88 µs | 11.88 ms | 11.57 ms | 3 |
| `(a+)+b`, 32 KiB of `a`, no match | 4.44 µs | 23.41 ms | 23.12 ms | 3 |
| `(a+)+b`, 64 KiB of `a`, no match | 4.49 µs | 47.04 ms | 46.52 ms | 3 |
| `[A-Za-z_][A-Za-z0-9_]*`, `example_123` | 5.24 µs | 1.75 µs | 0.72 µs | 1,000 |
| `[а-я]+`, `!!привет!!` | 2.49 µs | 1.68 µs | 0.81 µs | 1,000 |

For the identifier example, creating a fresh workspace for each search averaged
1.12 µs versus 0.72 µs when reusing one. The failing nested-repeat example grows
approximately with input size, rather than exponentially.
