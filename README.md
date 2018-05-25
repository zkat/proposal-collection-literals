# ECMAScript Tagged Collection Literals

## [Status](https://tc39.github.io/process-document/)

**Stage**: **WITHDRAWN**

**Author**: Kat Marchán (npm, [@maybekatz](https://twitter.com/maybekatz))

**Champions**: Kat Marchán (npm, [@maybekatz](https://twitter.com/maybekatz))

## WITHDRAWN

This proposal was presented to TC39 in May 2018 and voluntarily withdrawn during
discussions. The author concluded this issue was best solved through an
object-destructuring protocol corresponding to sequence iterators, in
combination with `Map.from()` and company.

JavaScript, unlike a lot of the static languages pattern matching concepts were
based on, does not include enough static typing information to make things like
destructuring or pattern matching efficient and cacheable enough to work well in
these cases and stay performant. Additionally, this additional syntax did not
add enough value to be worth the syntax budget expenditure.

A different proposal will be submitted based on this one that talks about
creation/extraction protocols in a unified way, but without adding additional
syntax.

## Introduction

This proposal extends both destructuring binding/assignment and the [`match`
statement](https://github.com/tc39/proposal-pattern-matching) with the ability
to apply custom destructuring and matching operations to matched data. It also
adds a new constructor syntax for building these custom data structures with
concise object and array style syntax while preserving their individual benefits
and invariants.

The syntax itself is derived from similar syntax in multiple other languages
used for this purpose, and is meant to be reminiscent of tagged template
literals in JavaScript -- except using other literal syntaxes available in the
language.

This proposal is derived from a previously-discussed [extensible collection
literals](https://github.com/alex-weej/es-extensible-collection-literal), but
adds significant work as far as how this syntax interacts with destructuring and
matching.

## Motivating Examples

Convenient construction of object-like and array-like data structures:
```js
const map = Map!{1: 2, three: 4, [[5]]: 6, 1: 'again'}
// Map { 1 => 'again', 'three' => 4, [5] => 6 }

const set = Set![1,2,3,2]
// Set [1, 2, 3]

const opt = Some!1
// Some { value: 1 }
```

Destructuring assignment/binding:
```js
const Map!{1: x, three: y} = map
x // 2
y // 4

const Set![x,y] = set
x // 1
y // 2

const Some!x = opt
x // 1
```

Match statement compatibility:
```js
match (input) {
  when Map!{1: x, 2: y} ~> ...
  when /(?<year>\d{4})-(?<month>\d{2})/u!{groups: {year, month}} ~> {
    console.log(`The year is ${year}, and the month is ${month}`)
  },
  when Some!1 ~> `option succeeded with an internal value of 1`
  when Some!x ~> `option succeeded with a non-1 value of ${x}`,
  when None!{} ~> `option failed`
}
```

## The Big Picture

### Related Active Proposals

* [Pattern matching](https://github.com/tc39/proposal-pattern-matching)
* [Frozen/sealed object syntax](https://github.com/keithamus/object-freeze-seal-syntax)
* [Richer Keys](https://docs.google.com/presentation/d/1q3CGeXqskL1gHTATH_VE9Dhj0VGTIAOzJ1cR0dYqDBk/edit#slide=id.p)
* [`Object.fromEntries`](https://github.com/bakkot/object-from-entries)
* [Smart Pipelines](https://github.com/js-choi/proposal-smart-pipelines/blob/master/readme.md)
* [`of` and `from` constructors](https://github.com/tc39/proposal-setmap-offrom)

## Construction Literals

For construction, literals are a thin layer of syntax sugar over
`Constructor.from()` functions. When a literal construction expression is found,
the left hand side is evaluated for its value, and the right hand side is
converted to an iterator or an atomic value. The type of value passed to
`.from()` depends on which of the three syntaxes is used:

```js
// Tagged Object Literals
Map!{foo: 1, 'foo': 1, [Symbol('bar')]: 2, 3: 4, [{}]: 5}
=== Map.from({[Symbol.iterator]: function* () {
  // IdentifierName interpreted as string
  yield ['foo', 1]
  // StringLiteral PropertyNames
  yield ['foo', 1]
  // Symbols preserved
  yield [Symbol('bar'), 2]
  // Numeric literals do not get ToString
  yield [3, 4]
  // Other kinds of computer property also do not get ToString
  yield [{}, 5]
}}

// Tagged Array Literals
Set![1,2,3]
=== Set.from({[Symbol.iterator]: function* () {
  // Nothing special here, except the argument is not an Array
  yield 1; yield 2; yield 3
}})

// Tagged Value Literals
Some!1
=== Some.from(1)
```

### Benefits

For Objects, the benefits are more obvious: No conversion to `ToString`,
parse-time early errors for invalid key/value syntax, and more appropriate
syntax for key/value types, instead of having to write nested arrays.

For Arrays and Values, the benefit on this end of things is smaller, and largely
based on convenience, with the exception that it does help ease a common footgun
with `new`:

```js
class Bar { constructor (iter) { this.val = [...iter] } }
function foo () { return {Bar} }
foo.Bar = Bar
new foo().Bar()
// TypeError: Class constructor Bar cannot be invoked without `new`
new foo.Bar()
// => Bar {}
```

While this behavior is consistent, it is something that does occasionally bite
people. Literal syntax helps ease this a bit, specially in data structure-heavy
code:

```js
foo().Bar![1,2,3]
// Bar { val: [1,2,3] }
foo.Bar![1,2,3]
// Bar { val: [1,2,3] }
```

But, as implied before, the main benefit of extending tagged literal syntax to
arrays and individual values is the correspondence to destructuring...

## Destructuring Literals

When a user learns they can construct with one syntax, is becomes much easier to
teach them how to destruct with it.

While the common `.from()` method mechanism is what makes construction literals
work, destructuring uses the standard iterator protocol through a
`Symbol.valueOf` constructor method. If `Symbol.valueOf` is not present,
`.valueOf()` is tried instead. If array or object-destructurng syntax is used
without RestProperty/RestElement, `.valueOf()` will receive an array of keys
that are being requested from the object. If an atomic destructure is requested,
the second argument to `valueOf()` will be `undefined`. Filtering entries based
on these keys is optional.

The `valueOf` method should return an iterator

```js
const Map!{1: x, y} = Map!{1: 'x', y: 'y'}
===
let x, y
for (let entry of Map.valueOf(Map!{1: 'x', y: 'y'}, [1, 'y'])) {
  match (entry) {
    when [1, _x] ~> {
      x = _x
    }
    when ['y', _y] ~> {
      y = _y
    }
  }
}

const Set![a, b, c] = Set![1,2,3,4]
===
let [a,b,c] = Array.from(Set.valueOf(Set![1,2,3,4], [0,1,2]))

class Some { constructor (val) { this._val = val } }
Some.valueOf = (some) => some._val

const Some!x = Some!1
===
let x = Some.valueOf(new Some(1))
// x === 1
```

When a destructuring sequence is used, the iterator will be used as-is to match
and fill entries in the destructured array or object.

If `valueOf` returns `undefined`, the match is considered to have failed, and no
values will match. When used with `match`, this will cause the match clause to
fail.
