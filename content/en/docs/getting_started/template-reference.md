---
title: "Template Reference"
draft: false
images: []
weight: 800
toc: true
---

Package Operator allows for in-cluster templating of files ending in `.gotmpl` via the [Go template engine](https://pkg.go.dev/text/template).

Package Operator templates use the [Masterminds Sprig](https://github.com/Masterminds/sprig) template library to offer additional functions. PKO templates aim to be reproducible, so Sprig template functions producing non-reproducible outputs, like current date/time, random number generation, etc. are not available.

- [Templates](#templates)
- [Dictionaries](#dictionaries)
- [String](#string)
- [String Slice](#string-slice)
- [Integer Slice](#integer-slice)
- [Integer Math](#integer-math)
- [Float Math](#float-math)
- [Defaults](#defaults)
- [Encoding](#encoding)
- [Lists](#lists)
- [Type Conversion](#type-conversion)
- [Cryptography](#cryptography)
- [Reflection](#reflection)
- [Path and Files](#path-and-files)
- [Semantic Version](#semantic-version)

## Templates

### `include`

The `include` function executes a pre-defined template and returns it as string, so it can be piped through over functions like `indent`.

```go
{{- define "test-helper" -}}test-helper{{- end -}}
{{include "include-test" . | upper | quote}}
---
"\"TEST-HELPER\""
```

## Dictionaries

Sprig provides a key/value storage type called a `dict` (short for "dictionary",
as in Python). A `dict` is an _unordered_ type.

The key to a dictionary **must be a string**. However, the value can be any
type, even another `dict` or `list`.

Unlike `list`s, `dict`s are not immutable. The `set` and `unset` functions will
modify the contents of a dictionary.

---

### `dict`

Creating dictionaries is done by calling the `dict` function and passing it a
list of pairs.

The following creates a dictionary with three items:

```go
$myDict := dict "name1" "value1" "name2" "value2" "name3" "value 3"
```

---

### `get`

Given a map and a key, get the value from the map.

```go
get $myDict "key1"
```

The above returns `"value1"`

Note that if the key is not found, this operation will simply return `""`. No error
will be generated.

---

### `set`

Use `set` to add a new key/value pair to a dictionary.

```go
$_ := set $myDict "name4" "value4"
```

Note that `set` _returns the dictionary_ (a requirement of Go template functions),
so you may need to trap the value as done above with the `$_` assignment.

---

### `unset`

Given a map and a key, delete the key from the map.

```go
$_ := unset $myDict "name4"
```

As with `set`, this returns the dictionary.

Note that if the key is not found, this operation will simply return. No error
will be generated.

---

### `hasKey`

The `hasKey` function returns `true` if the given dict contains the given key.

```go
hasKey $myDict "name1"
```

If the key is not found, this returns `false`.

---

### `pluck`

The `pluck` function makes it possible to give one key and multiple maps, and
get a list of all of the matches:

```go
pluck "name1" $myDict $myOtherDict
```

The above will return a `list` containing every found value (`[value1 otherValue1]`).

If the given key is _not found_ in a map, that map will not have an item in the
list (and the length of the returned list will be less than the number of dicts
in the call to `pluck`.

If the key is _found_ but the value is an empty value, that value will be
inserted.

A common idiom in Sprig templates is to uses `pluck... | first` to get the first
matching key out of a collection of dictionaries.

---

### `dig`

The `dig` function traverses a nested set of dicts, selecting keys from a list
of values. It returns a default value if any of the keys are not found at the
associated dict.

```go
dig "user" "role" "humanName" "guest" $dict
```

Given a dict structured like

```go
{
  user: {
    role: {
      humanName: "curator"
    }
  }
}
```

the above would return `"curator"`. If the dict lacked even a `user` field,
the result would be `"guest"`.

Dig can be very useful in cases where you'd like to avoid guard clauses,
especially since Go's template package's `and` doesn't shortcut. For instance
`and a.maybeNil a.maybeNil.iNeedThis` will always evaluate
`a.maybeNil.iNeedThis`, and panic if `a` lacks a `maybeNil` field.)

`dig` accepts its dict argument last in order to support pipelining. For instance:

```go
merge a b c | dig "one" "two" "three" "<missing>"
```

---

### `merge`, `mustMerge`

Merge two or more dictionaries into one, giving precedence to the dest dictionary:

```go
$newdict := merge $dest $source1 $source2
```

This is a deep merge operation but not a deep copy operation. Nested objects that
are merged are the same instance on both dicts. If you want a deep copy along
with the merge than use the `deepCopy` function along with merging. For example,

```go
deepCopy $source | merge $dest
```

`mustMerge` will return an error in case of unsuccessful merge.

---

### `mergeOverwrite`, `mustMergeOverwrite`

Merge two or more dictionaries into one, giving precedence from **right to left**, effectively
overwriting values in the dest dictionary:

Given:

```yaml
dst:
  default: default
  overwrite: me
  key: true

src:
  overwrite: overwritten
  key: false
```

will result in:

```yaml
newdict:
  default: default
  overwrite: overwritten
  key: false
```

```go
$newdict := mergeOverwrite $dest $source1 $source2
```

This is a deep merge operation but not a deep copy operation. Nested objects that
are merged are the same instance on both dicts. If you want a deep copy along
with the merge than use the `deepCopy` function along with merging. For example,

```go
deepCopy $source | mergeOverwrite $dest
```

`mustMergeOverwrite` will return an error in case of unsuccessful merge.

---

### `keys`

The `keys` function will return a `list` of all of the keys in one or more `dict`
types. Since a dictionary is _unordered_, the keys will not be in a predictable order.
They can be sorted with `sortAlpha`.

```go
keys $myDict | sortAlpha
```

When supplying multiple dictionaries, the keys will be concatenated. Use the `uniq`
function along with `sortAlpha` to get a unique, sorted list of keys.

```go
keys $myDict $myOtherDict | uniq | sortAlpha
```

---

### `pick`

The `pick` function selects just the given keys out of a dictionary, creating a
new `dict`.

```go
$new := pick $myDict "name1" "name2"
```

The above returns `{name1: value1, name2: value2}`

---

### `omit`

The `omit` function is similar to `pick`, except it returns a new `dict` with all
the keys that _do not_ match the given keys.

```go
$new := omit $myDict "name1" "name3"
```

The above returns `{name2: value2}`

---

### `values`

The `values` function is similar to `keys`, except it returns a new `list` with
all the values of the source `dict` (only one dictionary is supported).

```go
$vals := values $myDict
```

The above returns `list["value1", "value2", "value 3"]`. Note that the `values`
function gives no guarantees about the result ordering- if you care about this,
then use `sortAlpha`.

---

### `deepCopy`, `mustDeepCopy`

The `deepCopy` and `mustDeepCopy` functions takes a value and makes a deep copy
of the value. This includes dicts and other structures. `deepCopy` panics
when there is a problem while `mustDeepCopy` returns an error to the template
system when there is an error.

```go
dict "a" 1 "b" 2 | deepCopy
```

### A Note on Dict Internals

A `dict` is implemented in Go as a `map[string]interface{}`. Go developers can
pass `map[string]interface{}` values into the context to make them available
to templates as `dict`s.

## String

### `trim`

Removes whitespace from start and end of a string:

```go
trim "   hello   "
---
"hello"
```

---

### `trimAll`

Removes given characters from start and end of a string:

```go
trimAll "$" "$5.00"
---
"5.00"
```

---

### `trimSuffix`

Trim just the suffix from a string:

```go
trimSuffix "-" "hello-"
---
"hello"
```

---

### `trimPrefix`

Trim just the prefix from a string:

```go
trimPrefix "-" "-hello"
---
"hello"
```

---

### `upper`

Convert the entire string to uppercase:

```go
upper "hello"
---
"HELLO"
```

---

### `lower`

Convert the entire string to lowercase:

```go
lower "HELLO"
---
"hello"
```

---

### `title`

Convert to title case:

```go
title "hello world"
---
"Hello World"
```

---

### `untitle`

Remove title casing.

```go
untitle "Hello World"
---
"hello world"
```

---

### `repeat`

Repeat a string multiple times:

```go
repeat 3 "hello"
---
"hellohellohello"
```

---

### `substr`

Get a substring from a string. It takes three parameters:

- start (int)
- end (int)
- string (string)

```go
substr 0 5 "hello world"
---
"hello"
```

---

### `nospace`

Remove all whitespace from a string.

```go
nospace "hello w o r l d"
---
"helloworld"
```

---

### `trunc`

Truncate a string from the end:

```go
trunc 5 "hello world"
---
"hello"
```

Truncate a string from the beginning:

```go
trunc -5 "hello world"
---
"world"
```

---

### `abbrev`

Truncate a string with ellipses (`...`).

Parameters:

- max length
- the string

```
abbrev 5 "hello world"
---
"he..."
```

The above returns `he...`, since it counts the width of the ellipses against the
maximum length.

---

### `abbrevboth`

Abbreviate both sides.

Parameters:

- left offset
- max length
- the string

```go
abbrevboth 5 10 "1234 5678 9123"
---
"...5678..."
```

---

### `initials`

Given multiple words, take the first letter of each word and combine.

```go
initials "First Try"
---
"FT"
```

---

### `wrap`

Wrap text at a given column count:

```go
wrap 80 $someText
```

The above will wrap the string in `$someText` at 80 columns.

---

### `wrapWith`

`wrapWith` works as `wrap`, but lets you specify the string to wrap with.
(`wrap` uses `\n`)

```go
wrapWith 5 "\t" "Hello World"
---
"hello\tworld"
```

---

### `contains`

Test to see if one string is contained inside of another:

```go
contains "cat" "catch"
---
true
```

The above returns `true` because `catch` contains `cat`.

---

### `hasPrefix`, `hasSuffix`

The `hasPrefix` and `hasSuffix` functions test whether a string has a given
prefix or suffix:

```go
hasPrefix "cat" "catch"
---
true
```

The above returns `true` because `catch` has the prefix `cat`.

```go
hasSuffix "tch" "catch"
---
true
```

---

### `quote`, `squote`

These functions wrap a string in double quotes (`quote`) or single quotes
(`squote`).

```go
quote "hello"
---
"\"hello\""
```

```go
squote "hello"
---
"'hello'"
```

---

### `cat`

The `cat` function concatenates multiple strings together into one, separating
them with spaces:

```go
cat "hello" "beautiful" "world"
---
"hello beautiful world"
```

---

### `indent`

The `indent` function indents every line in a given string to the specified
indent width. This is useful when aligning multi-line strings:

```go
indent 4 $lots_of_text
```

The above will indent every line of text by 4 space characters.

---

### `nindent`

The `nindent` function is the same as the indent function, but prepends a new
line to the beginning of the string.

```go
nindent 4 $lots_of_text
```

The above will indent every line of text by 4 space characters and add a new
line to the beginning.

---

### `replace`

Perform simple string replacement.

It takes three arguments:

- string to replace
- string to replace with
- source string

```go
"I Am Henry VIII" | replace " " "-"
---
I-Am-Henry-VIII
```

---

### `plural`

Pluralize a string.

```go
len $fish | plural "one anchovy" "many anchovies"
```

In the above, if the length of the string is 1, the first argument will be
printed (`one anchovy`). Otherwise, the second argument will be printed
(`many anchovies`).

The arguments are:

- singular string
- plural string
- length integer

NOTE: Sprig does not currently support languages with more complex pluralization
rules. And `0` is considered a plural because the English language treats it
as such (`zero anchovies`). The Sprig developers are working on a solution for
better internationalization.

---

### `snakecase`

Convert string from camelCase to snake_case.

```go
snakecase "FirstName"
---
"first_name"
```

---

### `camelcase`

Convert string from snake_case to CamelCase

```go
camelcase "http_server"
---
"HttpServer"
```

---

### `kebabcase`

Convert string from camelCase to kebab-case.

```go
kebabcase "FirstName"
---
"first-name"
```

---

### `swapcase`

Swap the case of a string using a word based algorithm.

Conversion algorithm:

- Upper case character converts to Lower case
- Title case character converts to Lower case
- Lower case character after Whitespace or at start converts to Title case
- Other Lower case character converts to Upper case
- Whitespace is defined by unicode.IsSpace(char)

```go
swapcase "This Is A.Test"
---
"tHIS iS a.tEST"
```

---

### `regexMatch`, `mustRegexMatch`

Returns true if the input string contains any match of the regular expression.

```go
regexMatch "^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$" "test@acme.com"
---
true
```

`regexMatch` panics if there is a problem and `mustRegexMatch` returns an error to the
template engine if there is a problem.

---

### `regexFindAll`, `mustRegexFindAll`

Returns a slice of all matches of the regular expression in the input string.
The last parameter n determines the number of substrings to return, where -1 means return all matches

```go
regexFindAll "[2,4,6,8]" "123456789" -1
---
[2 4 6 8]
```

`regexFindAll` panics if there is a problem and `mustRegexFindAll` returns an error to the
template engine if there is a problem.

---

### `regexFind`, `mustRegexFind`

Return the first (left most) match of the regular expression in the input string

```go
regexFind "[a-zA-Z][1-9]" "abcd1234"
---
"d1"
```

`regexFind` panics if there is a problem and `mustRegexFind` returns an error to the
template engine if there is a problem.

---

### `regexReplaceAll`, `mustRegexReplaceAll`

Returns a copy of the input string, replacing matches of the Regexp with the replacement string replacement.
Inside string replacement, $ signs are interpreted as in Expand, so for instance $1 represents the text of the first submatch

```go
regexReplaceAll "a(x*)b" "-ab-axxb-" "${1}W"
---
"-W-xxW-"
```

`regexReplaceAll` panics if there is a problem and `mustRegexReplaceAll` returns an error to the
template engine if there is a problem.

---

### `regexReplaceAllLiteral`, `mustRegexReplaceAllLiteral`

Returns a copy of the input string, replacing matches of the Regexp with the replacement string replacement
The replacement string is substituted directly, without using Expand

```go
regexReplaceAllLiteral "a(x*)b" "-ab-axxb-" "${1}"
---
"-${1}-${1}-"
```

`regexReplaceAllLiteral` panics if there is a problem and `mustRegexReplaceAllLiteral` returns an error to the
template engine if there is a problem.

---

### `regexSplit`, `mustRegexSplit`

Slices the input string into substrings separated by the expression and returns a slice of the substrings between those expression matches. The last parameter `n` determines the number of substrings to return, where `-1` means return all matches

```go
regexSplit "z+" "pizza" -1
---
[pi a]
```

`regexSplit` panics if there is a problem and `mustRegexSplit` returns an error to the
template engine if there is a problem.

---

### `regexQuoteMeta`

Returns a string that escapes all regular expression metacharacters inside the argument text;
the returned string is a regular expression matching the literal text.

```go
regexQuoteMeta "1.2.3"
---
"1\.2\.3"
```

## String Slice

These function operate on or generate slices of strings. In Go, a slice is a
growable array. In Sprig, it's a special case of a `list`.

### `join`

Join a list of strings into a single string, with the given separator.

```go
list "hello" "world" | join "_"
---
`hello_world`
```

`join` will try to convert non-strings to a string value:

```go
list 1 2 3 | join "+"
---
"1+2+3"
```

---

### `splitList`, `split`

Split a string into a list of strings:

```go
splitList "$" "foo$bar$baz"
---
[foo bar baz]
```

The older `split` function splits a string into a `dict`. It is designed to make
it easy to use template dot notation for accessing members:

```go
$a := split "$" "foo$bar$baz"
```

The above produces a map with index keys. `{_0: foo, _1: bar, _2: baz}`

```
$a._0
```

The above produces `foo`

---

### `splitn`

`splitn` function splits a string into a `dict`. It is designed to make
it easy to use template dot notation for accessing members:

```go
$a := splitn "$" 2 "foo$bar$baz"
```

The above produces a map with index keys. `{_0: foo, _1: bar$baz}`

```go
$a._0
```

The above produces `foo`

---

### `sortAlpha`

The `sortAlpha` function sorts a list of strings into alphabetical (lexicographical)
order.

It does _not_ sort in place, but returns a sorted copy of the list, in keeping
with the immutability of lists.

## Integer Slice

### `until`

The `until` function builds a range of integers.

```go
until 5
```

The above generates the list `[0, 1, 2, 3, 4]`.

This is useful for looping with `range $i, $e := until 5`.

---

### `untilStep`

Like `until`, `untilStep` generates a list of counting integers. But it allows
you to define a start, stop, and step:

```go
untilStep 3 6 2
```

The above will produce `[3 5]` by starting with 3, and adding 2 until it is equal
or greater than 6. This is similar to Python's `range` function.

---

### `seq`

Works like the bash `seq` command.

- 1 parameter  (end) - will generate all counting integers between 1 and `end` inclusive.
- 2 parameters (start, end) - will generate all counting integers between `start` and `end` inclusive incrementing or decrementing by 1.
- 3 parameters (start, step, end) - will generate all counting integers between `start` and `end` inclusive incrementing or decrementing by `step`.

```
seq 5       => 1 2 3 4 5
seq -3      => 1 0 -1 -2 -3
seq 0 2     => 0 1 2
seq 2 -2    => 2 1 0 -1 -2
seq 0 2 10  => 0 2 4 6 8 10
seq 0 -2 -5 => 0 -2 -4
```

## Integer Math

### `add`

Sum numbers with `add`. Accepts two or more inputs.

```go
add 1 2 3
---
6
```

---

### `add1`

To increment by 1, use `add1`.

---

### `sub`

To subtract, use `sub`.

---

### `div`

Perform integer division with `div`.

---

### `mod`

Modulo with `mod`.

---

### `mul`

Multiply with `mul`. Accepts two or more inputs.

```go
mul 1 2 3
---
6
```

---

### `max`

Return the largest of a series of integers:

```go
max 1 2 3
---
3
```

---

### `min`

Return the smallest of a series of integers.

`min 1 2 3` will return `1`

## Float Math

### `addf`

Sum numbers with `addf`.

```go
addf 1.5 2 2
---
5.5
```

---

### `add1f`

To increment by 1, use `add1f`.

---

### `subf`

To subtract, use `subf`.

```go
subf 7.5 2 3
---
2.5
```

---

### `divf`

Perform integer division with `divf`.

This is equivalent to `10 / 2 / 4`:

```go
divf 10 2 4
---
1.25
```

---

### `mulf`

Multiply with `mulf`.

```go
mulf 1.5 2 2
---
6
```

---

### `maxf`

Return the largest of a series of floats:

```go
maxf 1 2.5 3
---
3
```

---

### `minf`

Return the smallest of a series of floats.

```go
minf 1.5 2 3
---
1.5
```

---

### `floor`

Returns the greatest float value less than or equal to input value

```go
floor 123.9999
---
123.0
```

---

### `ceil`

Returns the greatest float value greater than or equal to input value

```go
ceil 123.001
---
124.0
```

---

### `round`

Returns a float value with the remainder rounded to the given number to digits after the decimal point.

```go
round 123.555555 3
---
123.556
```

## Defaults

### `default`

To set a simple default value, use `default`:

```go
default "foo" .Bar
```

In the above, if `.Bar` evaluates to a non-empty value, it will be used. But if
it is empty, `foo` will be returned instead.

The definition of "empty" depends on type:

- Numeric: 0
- String: ""
- Lists: `[]`
- Dicts: `{}`
- Boolean: `false`
- And always `nil` (aka null)

For structs, there is no definition of empty, so a struct will never return the
default.

---

### `empty`

The `empty` function returns `true` if the given value is considered empty, and
`false` otherwise. The empty values are listed in the `default` section.

```go
empty .Foo
```

Note that in Go template conditionals, emptiness is calculated for you. Thus,
you rarely need `if empty .Foo`. Instead, just use `if .Foo`.

---

### `coalesce`

The `coalesce` function takes a list of values and returns the first non-empty
one.

```go
coalesce 0 1 2
---
1
```

This function is useful for scanning through multiple variables or values:

```go
coalesce .name .parent.name "Matt"
```

The above will first check to see if `.name` is empty. If it is not, it will return
that value. If it _is_ empty, `coalesce` will evaluate `.parent.name` for emptiness.
Finally, if both `.name` and `.parent.name` are empty, it will return `Matt`.

---

### `all`

The `all` function takes a list of values and returns true if all values are non-empty.

```go
all 0 1 2
---
false
```

This function is useful for evaluating multiple conditions of variables or values:

```go
all (eq .Request.TLS.Version 0x0304) (.Request.ProtoAtLeast 2 0) (eq .Request.Method "POST")
```

The above will check http.Request is POST with tls 1.3 and http/2.

---

### `any`

The `any` function takes a list of values and returns true if any value is non-empty.

```go
any 0 1 2
---
true
```

This function is useful for evaluating multiple conditions of variables or values:

```go
any (eq .Request.Method "GET") (eq .Request.Method "POST") (eq .Request.Method "OPTIONS")
```

The above will check http.Request method is one of GET/POST/OPTIONS.

---

### `ternary`

The `ternary` function takes two values, and a test value. If the test value is
true, the first value will be returned. If the test value is empty, the second
value will be returned. This is similar to the c ternary operator.

```go
ternary "foo" "bar" true
---
"foo"
```

or

```go
true | ternary "foo" "bar"
---
"foo"
```

## Encoding

### `fromYAML`

`fromYAML` decodes a YAML document into a structure.

```go
fromYAML $myYaml
```

### `toYAML`

`toYAML` encodes an item a YAML document.

### `fromJson`, `mustFromJson`

`fromJson` decodes a JSON document into a structure. If the input cannot be decoded as JSON the function will return an empty string.
`mustFromJson` will return an error in case the JSON is invalid.

```go
fromJson "{\"foo\": 55}"
```

---

### `toJson`, `mustToJson`

The `toJson` function encodes an item into a JSON string. If the item cannot be converted to JSON the function will return an empty string.
`mustToJson` will return an error in case the item cannot be encoded in JSON.

```go
toJson .Item
```

The above returns JSON string representation of `.Item`.

---

### `toPrettyJson`, `mustToPrettyJson`

The `toPrettyJson` function encodes an item into a pretty (indented) JSON string.

```go
toPrettyJson .Item
```

The above returns indented JSON string representation of `.Item`.

---

### `toRawJson`, `mustToRawJson`

The `toRawJson` function encodes an item into JSON string with HTML characters unescaped.

```go
toRawJson .Item
```

The above returns unescaped JSON string representation of `.Item`.

---

### `b64enc`, `b64dec`

Encode or decode with Base64.

---

### `b64decMap`

Base64 decode every value of the given map.

---

### `b32enc`/`b32dec`

Encode or decode with Base32.

## Lists

Sprig provides a simple `list` type that can contain arbitrary sequential lists
of data. This is similar to arrays or slices, but lists are designed to be used
as immutable data types.

Create a list of integers:

```go
$myList := list 1 2 3 4 5
```

The above creates a list of `[1 2 3 4 5]`.

---

### `first`, `mustFirst`

To get the head item on a list, use `first`.

`first $myList` returns `1`

`first` panics if there is a problem while `mustFirst` returns an error to the
template engine if there is a problem.

---

### `rest`, `mustRest`

To get the tail of the list (everything but the first item), use `rest`.

`rest $myList` returns `[2 3 4 5]`

`rest` panics if there is a problem while `mustRest` returns an error to the
template engine if there is a problem.

---

### `last`, `mustLast`

To get the last item on a list, use `last`:

`last $myList` returns `5`. This is roughly analogous to reversing a list and
then calling `first`.

---

### `initial`, `mustInitial`

This compliments `last` by returning all _but_ the last element.
`initial $myList` returns `[1 2 3 4]`.

`initial` panics if there is a problem while `mustInitial` returns an error to the
template engine if there is a problem.

---

### `append`, `mustAppend`

Append a new item to an existing list, creating a new list.

```go
$new = append $myList 6
```

The above would set `$new` to `[1 2 3 4 5 6]`. `$myList` would remain unaltered.

`append` panics if there is a problem while `mustAppend` returns an error to the
template engine if there is a problem.

---

### `prepend`, `mustPrepend`

Push an element onto the front of a list, creating a new list.

```go
prepend $myList 0
```

The above would produce `[0 1 2 3 4 5]`. `$myList` would remain unaltered.

`prepend` panics if there is a problem while `mustPrepend` returns an error to the
template engine if there is a problem.

---

### `concat`

Concatenate arbitrary number of lists into one.

```go
concat $myList ( list 6 7 ) ( list 8 )
```

The above would produce `[1 2 3 4 5 6 7 8]`. `$myList` would remain unaltered.

---

### `reverse`, `mustReverse`

Produce a new list with the reversed elements of the given list.

```go
reverse $myList
```

The above would generate the list `[5 4 3 2 1]`.

`reverse` panics if there is a problem while `mustReverse` returns an error to the
template engine if there is a problem.

---

### `uniq`, `mustUniq`

Generate a list with all of the duplicates removed.

```go
list 1 1 1 2 | uniq
```

The above would produce `[1 2]`

`uniq` panics if there is a problem while `mustUniq` returns an error to the
template engine if there is a problem.

---

### `without`, `mustWithout`

The `without` function filters items out of a list.

```go
without $myList 3
```

The above would produce `[1 2 4 5]`

Without can take more than one filter:

```go
without $myList 1 3 5
```

That would produce `[2 4]`

`without` panics if there is a problem while `mustWithout` returns an error to the
template engine if there is a problem.

---

### `has`, `mustHas`

Test to see if a list has a particular element.

```go
has 4 $myList
```

The above would return `true`, while `has "hello" $myList` would return false.

`has` panics if there is a problem while `mustHas` returns an error to the
template engine if there is a problem.

---

### `compact`, `mustCompact`

Accepts a list and removes entries with empty values.

``` go
$list := list 1 "a" "foo" ""
$copy := compact $list
```

`compact` will return a new list with the empty (i.e., "") item removed.

`compact` panics if there is a problem and `mustCompact` returns an error to the
template engine if there is a problem.

---

### `slice`, `mustSlice`

To get partial elements of a list, use `slice list [n] [m]`. It is
equivalent of `list[n:m]`.

- `slice $myList` returns `[1 2 3 4 5]`. It is same as `myList[:]`.
- `slice $myList 3` returns `[4 5]`. It is same as `myList[3:]`.
- `slice $myList 1 3` returns `[2 3]`. It is same as `myList[1:3]`.
- `slice $myList 0 3` returns `[1 2 3]`. It is same as `myList[:3]`.

`slice` panics if there is a problem while `mustSlice` returns an error to the
template engine if there is a problem.

---

### `chunk`

To split a list into chunks of given size, use `chunk size list`. This is useful for pagination.

```go
chunk 3 (list 1 2 3 4 5 6 7 8)
```

This produces list of lists `[ [ 1 2 3 ] [ 4 5 6 ] [ 7 8 ] ]`.

## Type Conversion

The following type conversion functions are provided by Sprig:

- `atoi`: Convert a string to an integer.
- `float64`: Convert to a `float64`.
- `int`: Convert to an `int` at the system's width.
- `int64`: Convert to an `int64`.
- `toDecimal`: Convert a unix octal to a `int64`.
- `toString`: Convert to a string.
- `toStrings`: Convert a list, slice, or array to a list of strings.

Only `atoi` requires that the input be a specific type. The others will attempt
to convert from any type to the destination type. For example, `int64` can convert
floats to ints, and it can also convert strings to ints.

---

### `toStrings`

Given a list-like collection, produce a slice of strings.

```go
list 1 2 3 | toStrings
```

The above converts `1` to `"1"`, `2` to `"2"`, and so on, and then returns
them as a list.

---

### `toDecimal`

Given a unix octal permission, produce a decimal.

```go
"0777" | toDecimal
```

The above converts `0777` to `511` and returns the value as an int64.

## Cryptography

### `sha1sum`

The `sha1sum` function receives a string, and computes it's SHA1 digest.

```
sha1sum "Hello world!"
```

---

### `sha256sum`

The `sha256sum` function receives a string, and computes it's SHA256 digest.

```
sha256sum "Hello world!"
```

The above will compute the SHA 256 sum in an "ASCII armored" format that is
safe to print.

---

### `adler32sum`

The `adler32sum` function receives a string, and computes its Adler-32 checksum.

```
adler32sum "Hello world!"
```

## Reflection

Sprig provides rudimentary reflection tools. These help advanced template
developers understand the underlying Go type information for a particular value.

Go has several primitive _kinds_, like `string`, `slice`, `int64`, and `bool`.

Go has an open _type_ system that allows developers to create their own types.

Sprig provides a set of functions for each.

---

### Kind

There are two Kind functions: `kindOf` returns the kind of an object.

```
kindOf "hello"
```

The above would return `string`. For simple tests (like in `if` blocks), the
`kindIs` function will let you verify that a value is a particular kind:

```
kindIs "int" 123
```

The above will return `true`

---

### Type

Types are slightly harder to work with, so there are three different functions:

- `typeOf` returns the underlying type of a value: `typeOf $foo`
- `typeIs` is like `kindIs`, but for types: `typeIs "*io.Buffer" $myVal`
- `typeIsLike` works as `typeIs`, except that it also dereferences pointers.

**Note:** None of these can test whether or not something implements a given
interface, since doing so would require compiling the interface in ahead of time.

---

### `deepEqual`

`deepEqual` returns true if two values are ["deeply equal"](https://golang.org/pkg/reflect/#DeepEqual)

Works for non-primitive types as well (compared to the built-in `eq`).

```go
deepEqual (list 1 2 3) (list 1 2 3)
```

The above will return `true`

## Path and Files

While Sprig does not grant access to the filesystem, it does provide functions
for working with strings that follow file path conventions.

### Paths

Paths separated by the slash character (`/`), processed by the `path` package.

Examples:

- The [Linux](https://en.wikipedia.org/wiki/Linux) and
  [MacOS](https://en.wikipedia.org/wiki/MacOS)
  [filesystems](https://en.wikipedia.org/wiki/File_system):
  `/home/user/file`, `/etc/config`;
- The path component of
  [URIs](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier):
  `https://example.com/some/content/`, `ftp://example.com/file/`.

---

### `base`

Return the last element of a path.

```go
base "foo/bar/baz"
```

The above prints "baz".

---

### `dir`

Return the directory, stripping the last part of the path. So `dir "foo/bar/baz"`
returns `foo/bar`.

---

### `clean`

Clean up a path.

```go
clean "foo/bar/../baz"
```

The above resolves the `..` and returns `foo/baz`.

---

### `ext`

Return the file extension.

```go
ext "foo.bar"
```

The above returns `.bar`.

---

### `isAbs`

To check whether a path is absolute, use `isAbs`.

---

### `getFile`

Access another file in the package. The file's content will be available as string and can be piped to other functions.

```
{{ getFile "_stuff.txt" }}
```

## Semantic Version

Some version schemes are easily parseable and comparable. Sprig provides functions
for working with [SemVer 2](http://semver.org) versions.

---

### `semver`

The `semver` function parses a string into a Semantic Version:

```
$version := semver "1.2.3-alpha.1+123"
```

_If the parser fails, it will cause template execution to halt with an error._

At this point, `$version` is a pointer to a `Version` object with the following
properties:

- `$version.Major`: The major number (`1` above)
- `$version.Minor`: The minor number (`2` above)
- `$version.Patch`: The patch number (`3` above)
- `$version.Prerelease`: The prerelease (`alpha.1` above)
- `$version.Metadata`: The build metadata (`123` above)
- `$version.Original`: The original version as a string

Additionally, you can compare a `Version` to another `version` using the `Compare`
function:

```
semver "1.4.3" | (semver "1.2.3").Compare
```

The above will return `-1`.

The return values are:

- `-1` if the given semver is greater than the semver whose `Compare` method was called
- `1` if the version who's `Compare` function was called is greater.
- `0` if they are the same version

(Note that in SemVer, the `Metadata` field is not compared during version
comparison operations.)

---

### `semverCompare`

A more robust comparison function is provided as `semverCompare`. It returns `true` if
the constraint matches, or `false` if it does not match. This version supports version ranges:

- `semverCompare "1.2.3" "1.2.3"` checks for an exact match
- `semverCompare "^1.2.0" "1.2.3"` checks that the major and minor versions match, and that the patch
  number of the second version is _greater than or equal to_ the first parameter.

The SemVer functions use the [Masterminds semver library](https://github.com/Masterminds/semver),
from the creators of Sprig.

### Basic Comparisons

There are two elements to the comparisons. First, a comparison string is a list
of space or comma separated AND comparisons. These are then separated by || (OR)
comparisons. For example, `">= 1.2 < 3.0.0 || >= 4.2.3"` is looking for a
comparison that's greater than or equal to 1.2 and less than 3.0.0 or is
greater than or equal to 4.2.3.

The basic comparisons are:

- `=`: equal (aliased to no operator)
- `!=`: not equal
- `>`: greater than
- `<`: less than
- `>=`: greater than or equal to
- `<=`: less than or equal to

_Note, according to the Semantic Version specification pre-releases may not be
API compliant with their release counterpart. It says,_

### Working With Prerelease Versions

Pre-releases, for those not familiar with them, are used for software releases
prior to stable or generally available releases. Examples of prereleases include
development, alpha, beta, and release candidate releases. A prerelease may be
a version such as `1.2.3-beta.1` while the stable release would be `1.2.3`. In the
order of precedence, prereleases come before their associated releases. In this
example `1.2.3-beta.1 < 1.2.3`.

According to the Semantic Version specification prereleases may not be
API compliant with their release counterpart. It says,

> A pre-release version indicates that the version is unstable and might not satisfy the intended compatibility requirements as denoted by its associated normal version.

SemVer comparisons using constraints without a prerelease comparator will skip
prerelease versions. For example, `>=1.2.3` will skip prereleases when looking
at a list of releases while `>=1.2.3-0` will evaluate and find prereleases.

The reason for the `0` as a pre-release version in the example comparison is
because pre-releases can only contain ASCII alphanumerics and hyphens (along with
`.` separators), per the spec. Sorting happens in ASCII sort order, again per the
spec. The lowest character is a `0` in ASCII sort order
(see an [ASCII Table](http://www.asciitable.com/))

Understanding ASCII sort ordering is important because A-Z comes before a-z. That
means `>=1.2.3-BETA` will return `1.2.3-alpha`. What you might expect from case
sensitivity doesn't apply here. This is due to ASCII sort ordering which is what
the spec specifies.

### Hyphen Range Comparisons

There are multiple methods to handle ranges and the first is hyphens ranges.
These look like:

- `1.2 - 1.4.5` which is equivalent to `>= 1.2 <= 1.4.5`
- `2.3.4 - 4.5` which is equivalent to `>= 2.3.4 <= 4.5`

### Wildcards In Comparisons

The `x`, `X`, and `*` characters can be used as a wildcard character. This works
for all comparison operators. When used on the `=` operator it falls
back to the patch level comparison (see tilde below). For example,

- `1.2.x` is equivalent to `>= 1.2.0, < 1.3.0`
- `>= 1.2.x` is equivalent to `>= 1.2.0`
- `<= 2.x` is equivalent to `< 3`
- `*` is equivalent to `>= 0.0.0`

### Tilde Range Comparisons (Patch)

The tilde (`~`) comparison operator is for patch level ranges when a minor
version is specified and major level changes when the minor number is missing.
For example,

- `~1.2.3` is equivalent to `>= 1.2.3, < 1.3.0`
- `~1` is equivalent to `>= 1, < 2`
- `~2.3` is equivalent to `>= 2.3, < 2.4`
- `~1.2.x` is equivalent to `>= 1.2.0, < 1.3.0`
- `~1.x` is equivalent to `>= 1, < 2`

### Caret Range Comparisons (Major)

The caret (`^`) comparison operator is for major level changes once a stable
(1.0.0) release has occurred. Prior to a 1.0.0 release the minor versions acts
as the API stability level. This is useful when comparisons of API versions as a
major change is API breaking. For example,

- `^1.2.3` is equivalent to `>= 1.2.3, < 2.0.0`
- `^1.2.x` is equivalent to `>= 1.2.0, < 2.0.0`
- `^2.3` is equivalent to `>= 2.3, < 3`
- `^2.x` is equivalent to `>= 2.0.0, < 3`
- `^0.2.3` is equivalent to `>=0.2.3 <0.3.0`
- `^0.2` is equivalent to `>=0.2.0 <0.3.0`
- `^0.0.3` is equivalent to `>=0.0.3 <0.0.4`
- `^0.0` is equivalent to `>=0.0.0 <0.1.0`
- `^0` is equivalent to `>=0.0.0 <1.0.0`
