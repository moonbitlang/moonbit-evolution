# Lexmatch expression

- Proposal:
  [ME-0011](https://github.com/moonbitlang/moonbit-evolution/blob/0011-lexmatch-expression/proposals/0011-lexmatch-expression.mbt.md)
- Author: [Wen Yuxiang](https://github.com/hackwaly)
- Status: Experimental
- Review and discussion: [GitHub
  issue](https://github.com/moonbitlang/moonbit-evolution/pull/15)

## Introduction

MoonBit aims to be perfect for data processing tasks. Currently, the `match`
expression is the primary way to destructure and analyze data. However, it is
not as powerful as Regular Expressions in string processing. This proposal
introduces a new expression called `lexmatch`, which combines the capabilities
of `match` and Regular Expressions to provide a more flexible and powerful way
to analyze and destructure strings.

## Examples

### Word count

The following function counts the number of lines, words, and characters in a
given input string. It uses the `lexmatch` expression to match different
patterns in the input string.

```moonbit
///|
pub fn wordcount(
  input : BytesView,
  lines : Int,
  words : Int,
  chars : Int,
) -> (Int, Int, Int) {
  lexmatch input with longest {
    ("\n", rest) => wordcount(rest, lines + 1, words, chars)
    ("[^ \t\r\n]+" as word, rest) =>
      wordcount(rest, lines, words + 1, chars + word.length())
    (".", rest) => wordcount(rest, lines, words, chars + 1)
    "" => (lines, words, chars)
    _ => panic()
  }
}
```

### Downloadable protocol extractor

The following function extracts the protocol from a given URL if it is a
downloadable protocol (ftp, http, https). It demonstrates the use of `lexmatch?`
expression and case-insensitive modifier.

```moonbit
///|
pub fn downloadable_protocol(url: StringView) -> StringView? {
  if url lexmatch? (("(?i:ftp|http(s)?)" as protocol) "://", _) with longest {
    Some(protocol)
  } else {
    None
  }
}
```

### Search for first block comment

The following function searches for the first block comment in a given string.
It demonstrates the use of `lexmatch` expression with `first` match strategy,
search mode, and non-greedy quantifiers.

```moonbit
///|
pub fn first_block_comment(str : StringView) -> StringView? {
  lexmatch str {
    (_, "/\*" (".*?" as content) "\*/", _) => Some(content)
    _ => None
  }
}
```

## Explanation

### Terminology

- **Target**: The `StringView` or `BytesView` to be `lexmatch`ed.
- **Match Strategy**: The strategy used to match patterns. It can be either
  `longest` or `first` (default).

- **Catch-all case**: A case with a variable or wildcard `_` as its left-hand
  side, which matches any target. It is required to be placed at the end of the
  `lexmatch` arms/cases to handle unmatched situations.
- **Lex Pattern**: The pattern part (differ with guard part) in left-hand side
  of a `lexmatch` arm/case (before `=>`).

  E.g. `("[^ \t\r\n]+" as word, rest)`

  A lex pattern can be one of the following:

  - Bare regex pattern: the regex pattern will match against the whole target.

    E.g. `""`

    In this case, the regex pattern is `""`, which matches an empty `StringView`
    or `BytesView`.

  - Regex pattern followed by a comma and a rest variable: the regex pattern
    will match against the prefix of the target, and the rest variable will bind
    to the remaining suffix.

    The rest variable can be either a variable or a wildcard `_`.

    In this form, the parentheses are required to improve readability.

    E.g. `("\n", rest)`

    In this case, the regex pattern is `"\n"`, which matches a newline character
    at the beginning of the target. The `rest` variable will bind to the
    remaining suffix of the target after removing the matched prefix.

  - Wildcard `_` followed by a comma, a regex pattern, another comma, and a rest
    variable. This so called "search mode" form, and can only be used under
    "first" match strategy.
    
    The regex pattern will search for the first match in the
    target. The `_` matches the prefix before the match, and the rest variable
    will bind to the remaining suffix after the match.

    The rest variable can be either a variable or a wildcard `_`.

    E.g. `(_, "/\*" (".*?" as content) "\*/", _)`

    In this case, the regex pattern is `"/\\*" (".*?" as content) "\\*/"`, which
    matches a block comment. The `_` matches the prefix before the block
    comment, and the last `_` matches the remaining suffix after the block
    comment.

- **Regex Pattern**: Regex patterns have three forms:

  - **Regex Literal**: A string literal representing a regex pattern.

    E.g. `"[^ \t\r\n]+"`

  - **Capture**: A regex pattern followed by `as` and a variable name to capture
    the matched substring.

    E.g. `"[^ \t\r\n]+" as word`

    If the lex pattern is a bare regex pattern of this form, the parentheses are
    required.

  - **Sequence**: A sequence of regex patterns separated by whitespace.

    E.g. `"//" ("[^\r\n]*" as comment)`

    If the lex pattern is a bare regex pattern of this form, the parentheses are
    required.

  Regex patterns can be nested to form more complex patterns.

### Semantics

The `lexmatch` expression works similarly to the `match` expression, with the
following differences:

1. The target of a `lexmatch` expression must be a `StringView` or `BytesView`.
2. Each arm/case except the catch-all case of a `lexmatch` expression must have
   a lex pattern as its left-hand side.
3. The match strategy can be specified after the `with` keyword. If not
   specified, the default strategy is `first`.
4. The regex patterns in lex patterns are matched against the target using the
   specified match strategy.
5. If a regex pattern matches the target, any capture variables in the pattern
   will be bound to the corresponding matched substrings.
6. If a regex pattern followed by a comma and a rest variable matches the
   target, the regex pattern will match the prefix of the target, and the rest
   variable will bind to the remaining suffix.
7. If a wildcard `_` followed by a comma, a regex pattern, another comma, and a
   rest variable matches the target, the regex pattern will search for the first
   match in the target. The `_` matches the prefix before the match, and the
   rest variable will bind to the remaining suffix.
8. If no lex pattern matches the target, the catch-all case will be executed.

### Subtleties

- When capture a single character, the matched substring is a `Char` or `Byte`,
  instead of a `StringView` or `BytesView`. E.g. `("[+-]" as sign)`

- The `"(abc)"` regex pattern does not introduce a capture group. To capture the
  matched substring, you need to use the `as` syntax. E.g. `"abc" as group`
  instead of `"(abc)"`.

- The `"$"` regex pattern matches the end of the target `StringView` or
  `BytesView`. The `"^"` regex pattern matches the start of the target (not
  implemented for now).

- The scoped modifiers syntax, such as `(?i:...)`, can only enable one modifier
  per group. E.g. `(?im:...)` is not supported for now. And doesn't support
  negating modifiers, such as `(?-i:...)`. For now, the only supported modifier
  is `i` (case-insensitive).

- Non-greedy quantifiers (e.g. `*?`, `+?`, `??`, `{n,m}?`) are supported under
  first match strategy.

### Special Character classes

The following special character classes are supported in regex patterns:

- `.`: Matches any single character.
- `\s`: Matches any whitespace character in ASCII range. Equivalent to `[
  \t\r\n\f\v]`
- `\S`: Matches any non-whitespace character in ASCII range. Equivalent to `[^
  \t\r\n\f\v]`
- `\d`: Matches any digit character in ASCII range. Equivalent to `[0-9]`
- `\D`: Matches any non-digit character in ASCII range. Equivalent to `[^0-9]`
- `\w`: Matches any word character in ASCII range. Equivalent to `[a-zA-Z0-9_]`
- `\W`: Matches any non-word character in ASCII range. Equivalent to
  `[^a-zA-Z0-9_]`

## FAQ

- Why not use the `match` expression with regex patterns directly?

  The `match` expression is designed for structural pattern matching, while the
  `lexmatch` expression is designed for lexical analysis. Mixing the two
  concepts may lead to confusion and complexity. By introducing a separate
  expression for lexical analysis, we can keep the semantics clear and focused.

- Which syntaxes/features can be used in regex literals currently?

  Bascially, the syntax aligned with JavaScript regex literals (with v flag
  enabled).

- What features might be supported in the future?

  - Regex flags (e.g. `i`, `g`, `m`, etc.)
  - Lookahead and lookbehind assertions (e.g. `(?=...)`, `(?!...)`, etc.)
  - Backreferences (e.g. `\1`, `\2`, etc.)
  - Named capture groups (e.g. `(?<name>...)`)
  - Unicode property escapes (e.g. `\p{...}`, `\P{...}`)
  - Scoped modifiers (e.g. `(?i:...)`, `(?-i:...)`)
