# Ghi validation

Small, explicit input validation for Ghi. Package identity: `arm092/validation`.
Source namespace: `arm092.validation`. MIT licensed. No external dependencies,
reflection, struct tags, automatic conversion, or global state.

## Install

```sh
mojave add arm092/validation https://github.com/arm092/ghi-validation.git v0.1.0
```

Verified with the published Windows amd64 Ghi v0.2.5 and Mojave v0.1.0.
Other compiler versions and platforms have not been verified.

## Use

Create a new validator for each input. Checks accumulate violations in call
order and do not change the supplied values.

```ghi
import arm092.validation.Validator
import math "go:math"
import strings "go:strings"

check := new Validator()
check.RequiredText("title", title)
check.TextLength("title", title, 1, 200)
check.OneOf("status", status, []string{"open", "in_progress", "resolved"})
check.IntRange("limit", limit, 1, 100)
check.IntRange("offset", offset, 0, math.MaxInt)
if strings.ContainsRune(title, 0) {
    check.Reject("title", "nul_character")
}

if !check.Valid() {
    for _, violation := range check.Errors() {
        println(violation.Field(), violation.Code())
    }
}
```

The application decides how to map codes to messages, HTTP responses, or
localized text. An optional status filter can skip `OneOf` when its value is
empty. If the application wants trimmed titles, trim them explicitly before
these checks.

## Rules

| Method | Meaning | Violation codes |
| --- | --- | --- |
| `RequiredText(field, value)` | Reject empty or Unicode-whitespace-only text. Does not trim the value. | `required` |
| `TextLength(field, value, minimum, maximum)` | Inclusive Unicode code point length; rejects invalid UTF-8. | `min_length`, `max_length`, `invalid_utf8` |
| `OneOf(field, value, allowed)` | Exact, case-sensitive string membership. An empty value is allowed when explicitly present in `allowed`. | `one_of` |
| `IntRange(field, value, minimum, maximum)` | Inclusive native `int` bounds, without conversion to floating point. | `min_value`, `max_value` |
| `NumberRange(field, value, minimum, maximum)` | Inclusive finite `float64` bounds. Rejects NaN and infinities. | `min_value`, `max_value`, `invalid_number` |
| `Reject(field, code)` | Add an explicit application-specific violation. | Caller-provided code |

`TextLength` counts code points, not bytes or user-perceived grapheme clusters.
For example, `🙂` has length 1 and `e` followed by a combining accent has length
2. No Unicode normalization is performed. Length includes whitespace.
`RequiredText` only checks emptiness; use `TextLength` to validate UTF-8 too.

Rules are independent. Calling both `RequiredText` and `TextLength` may report
both `required` and `min_length` for one field. Repeated failures are retained,
including duplicate field/code pairs.

## Results and configuration errors

`Valid()` reports whether there are no violations. `Errors()` returns all
violations; `ForField(field)` returns those for an exact field name. Both
return independent slices. A `Violation` exposes read-only `Field()` and
`Code()` methods. Import `arm092.validation.Violation` when an explicit type
annotation is needed.

Invalid rule configuration throws `arm092.validation.ConfigError` instead of
adding a user-input violation. Field names must not be blank, text length bounds
must satisfy `0 <= minimum <= maximum`, range bounds must be ordered, floating
bounds must be finite, allowed values must not be empty, and custom codes must
not be blank. Catch this exception only when configuration is dynamic.

A validator is mutable and intended for one payload in one goroutine. Create
a new instance for the next payload; there is no shared registry or reset.

## Tests

```sh
ghi fmt --check .
ghi test -v .
```

Tests live under `tests/`. They cover Unicode and invalid UTF-8, inclusive
bounds, non-finite numbers, ordered aggregation, field grouping, snapshot
isolation, and invalid configuration.
