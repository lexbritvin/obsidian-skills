# DQL functions

Every function callable from DQL. All also available in DataviewJS via `dv.func.<name>`. Each entry: signature → one-line example.

## 1. Constructors / type coercion

- `object(key1, value1, key2, value2, ...)` → `object("a", 6) = {a: 6}`.
- `list(value1, value2, ...)` (alias `array(...)`) → `list(1, 2, 3)`.
- `date(any)` → `date("2020-04-18")`. Special tokens: `today`, `now`, `tomorrow`, `yesterday`, `sow`, `eow`, `som`, `eom`, `soy`, `eoy`.
- `date(text, format)` (Luxon tokens) → `date("12/31/2022", "MM/dd/yyyy")`.
- `dur(any)` → `dur("8 minutes, 4 seconds")`. Units: `s/sec/second(s)`, `m/min/minute(s)`, `h/hr/hour(s)`, `d/day(s)`, `w/wk/week(s)`, `mo/month(s)`, `yr/year(s)`.
- `number(string)` → `number("18 years") = 18`. Extracts the leading numeric.
- `string(any)` → `string(18) = "18"`.
- `link(path, [display])` → `link("Hello", "Goodbye")`.
- `embed(link, [embed?])` → `embed(link("Hello.png"))`. Returns a link with the `embed` flag toggled.
- `elink(url, [display])` → `elink("www.google.com", "Google")`. External link.
- `typeof(any)` → returns one of `"number"`, `"string"`, `"boolean"`, `"date"`, `"duration"`, `"link"`, `"array"`, `"object"`, `"null"`. Example: `typeof([1,2,3]) = "array"`.

## 2. Numeric

- `round(number, [digits])` → `round(16.555555, 2) = 16.56`. `digits` defaults to 0.
- `trunc(number)` → `trunc(12.937) = 12`.
- `floor(number)` → `floor(12.937) = 12`.
- `ceil(number)` → `ceil(12.937) = 13`.
- `abs(number)` → `abs(-3) = 3`.
- `min(a, b, ...)` (variadic) → `min(1, 2, 3) = 1`. Also accepts a single list: `min(list)`.
- `max(a, b, ...)` → `max(1, 2, 3) = 3`. Also `max(list)`.
- `sum(array)` → `sum([1, 2, 3]) = 6`.
- `product(array)` → `product([1, 2, 3]) = 6`.
- `average(array)` → `average([1, 2, 3]) = 2`.
- `reduce(array, operand)` → `reduce([100, 20, 3], "-") = 77`. Operand: `"+"`, `"-"`, `"*"`, `"/"`, `"&"`, `"|"`.
- `minby(array, fn)` → `minby([1, 2, 3], (k) => k) = 1`.
- `maxby(array, fn)` → `maxby([1, 2, 3], (k) => k) = 3`.

## 3. String

- `regextest(pattern, string)` (substring match) → `regextest("\\w+", "hello") = true`.
- `regexmatch(pattern, string)` (full-string match) → `regexmatch("\\w+", "hello") = true`.
- `regexreplace(string, pattern, replacement)` → `regexreplace("yes", "[ys]", "a") = "aea"`.
- `replace(string, pattern, replacement)` (literal substring, all occurrences) → `replace("what", "wh", "h") = "hat"`.
- `lower(string)` → `lower("Test") = "test"`.
- `upper(string)` → `upper("Test") = "TEST"`.
- `split(string, delimiter, [limit])` → `split("hello world", " ") = list("hello", "world")`.
- `startswith(string, prefix)` → `startswith("yes", "ye") = true`.
- `endswith(string, suffix)` → `endswith("yes", "es") = true`.
- `padleft(string, length, [padding])` → `padleft("hello", 7) = "  hello"`. Default pad char is space.
- `padright(string, length, [padding])` → `padright("hello", 7) = "hello  "`.
- `substring(string, start, [end])` → `substring("hello", 0, 2) = "he"`.
- `truncate(string, length, [suffix])` → `truncate("Hello there!", 8) = "Hello..."`. Default suffix `"..."`.

## 4. List / object / containment

- `contains(any, value)` (case-sensitive) → `contains("Hello", "lo") = true`. Works for strings, lists, and objects (key match).
- `icontains(any, value)` (case-insensitive substring) → `icontains("Hello", "Lo") = true`.
- `econtains(any, value)` (exact match within a list / strict substring within a string) → `econtains(["yes", "no"], "yes") = true`.
- `containsword(string|list, value)` (whole word, case-insensitive) → `containsword("My word here", "word") = true`.
- `extract(object, key1, key2, ...)` → `extract(file, "ctime", "mtime")`. Returns a sub-object containing only those keys.
- `sort(list)` → `sort(list(3, 2, 1)) = list(1, 2, 3)`.
- `reverse(list)` → `reverse(list(1, 2, 3)) = list(3, 2, 1)`.
- `length(any)` → `length([1, 2, 3]) = 3`. Works for strings, lists, and objects (key count).
- `nonnull(array)` → `nonnull([null, false, 1]) = [false, 1]`.
- `firstvalue(array)` → `firstvalue([null, 1, 2]) = 1`.
- `all(array)` / `all(array, fn)` → `all([1, 2, 3]) = true`. With predicate: every element matches.
- `any(array)` / `any(array, fn)` → `any([0, 1]) = true`. With predicate: some element matches.
- `none(array)` / `none(array, fn)` → `none([]) = true`. With predicate: no element matches.
- `join(array, [delimiter])` → `join(list(1, 2, 3)) = "1, 2, 3"`. Default delimiter `", "`.
- `filter(array, predicate)` → `filter([1, 2, 3], (x) => x >= 2) = [2, 3]`.
- `unique(array)` → `unique([1, 3, 7, 3, 1]) = [1, 3, 7]`.
- `map(array, fn)` → `map([1, 2, 3], (x) => x + 2) = [3, 4, 5]`.
- `flat(array, [depth])` → `flat(list(1, 2, list(3, 4)), 1) = list(1, 2, 3, 4)`. Default depth `1`.
- `slice(array, [start, [end]])` → `slice([1, 2, 3, 4, 5], 3) = [4, 5]`.

## 5. Date

- `date(any)` — see § 1.
- `dateformat(date|datetime, formatString)` (Luxon tokens) → `dateformat(file.ctime, "yyyy-MM-dd") = "2024-05-08"`.
  - Common tokens: `yyyy`, `yy`, `LLLL` (month name), `MM`, `MMM`, `dd`, `EEEE` (weekday), `HH`, `mm`, `ss`.
- `durationformat(duration, formatString)` → `durationformat(dur("3 days 7 hours 43 seconds"), "ddd'd' hh'h' ss's'") = "003d 07h 43s"`.
- `striptime(date)` → `striptime(file.ctime) = file.cday`. Drops the time component.
- `localtime(date)` → converts a fixed-zone date to the local timezone.
- `currencyformat(number, [currency])` (ISO 4217) → `currencyformat(123456.789, "EUR") = "€123,456.79"`.

Date arithmetic is built-in (no function needed):

- `date - date` → `Duration`.
- `date ± duration` → `date`.

## 6. Utility / formatting / meta

- `default(field, value)` → `default(deadline, "none")`. Returns `value` when `field` is `null`.
- `ldefault(field, value)` (list-aware default) — returns `[value]` if the field is null, else the field. Useful before applying list operators.
- `display(any)` → `display(link("path/to/file.md")) = "file"`. Renders the visible text of a value.
- `choice(bool, left, right)` → `choice(true, "yes", "no") = "yes"`. Ternary expression.
- `hash(seed, [text], [variant])` → `hash(dateformat(date(today), "yyyy-MM-dd"), file.name)`. Deterministic hash, useful for "daily-stable" sampling.
- `meta(link)` — exposes link metadata (returns an object with the fields below; access via `.field`):
  - `.path` — `meta([[My Project]]).path = "My Project"`.
  - `.subpath` — `meta([[My Project#Notes]]).subpath = "Notes"`.
  - `.display` — `meta([[Page|Display Text]]).display = "Display Text"`.
  - `.embed` — boolean, `true` for `![[…]]`.
  - `.type` — `"file"`, `"header"`, or `"block"`.

## 7. Calling functions from DataviewJS

Every function above is bound on `dv.func`:

```js
dv.func.length(dv.current().file.tasks);
dv.func.dateformat(dv.date("2026-05-08"), "yyyy-MM-dd");
dv.func.contains(["a", "b"], "a");
dv.func.regexreplace("yes", "[ys]", "a");
dv.func.choice(x > 0, "positive", "non-positive");
dv.func.meta(dv.fileLink("Notes/Index", false, "Index")).display;
```

## 8. Vectorisation

Operators and many functions broadcast over lists element-wise:

- `[1, 2, 3] + 1 = [2, 3, 4]`
- `lower(["YES", "NO"]) = ["yes", "no"]`
- `length(["a", "bc"]) = [1, 2]`

## 9. Pitfalls

- **`regexmatch` vs `regextest`** — `regexmatch` requires the pattern to match the **whole** string; `regextest` allows substring matches. Forgetting this is the most common source of "why is my regex failing" questions.
- **`contains` vs `icontains` vs `econtains`** — `contains` is case-sensitive substring, `icontains` is case-insensitive substring, `econtains` is exact match (full-string for strings, exact element for lists).
- **Variadic `min`/`max`** also accept a list: `min(rows.rating)`, `max(rows.rating)`. Don't double-wrap as `min(list(rows.rating))`.
- **`reduce` operand is a string** — `reduce(arr, "+")`, not `reduce(arr, +)`.
- **`flat` default depth is 1** — pass a number to flatten further. Use `Infinity`-style: it doesn't accept `infinity`; if you need fully recursive flattening, prefer `expand(...)` in DataviewJS.
- **`dateformat` uses Luxon tokens, not Moment** — `yyyy` (year, lower case in Luxon) is correct; `YYYY` is local-week-year and rarely what you want.
- **`date(text, format)` is for parsing** non-ISO inputs. With ISO inputs the second arg is unnecessary.
- **`default` does not coerce empty strings or empty arrays** — only `null` triggers the fallback. Use `length(field) = 0` for empty-list checks.
