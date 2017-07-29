# Extensible Collection Literal

This is a stage 0, "strawman" proposal for the addition of an extensible collection literal syntax to the ECMAScript language.

## Proposal Overview

Given:

```js
const IList = Immutable.List;
const me = "Alexander";
```

then

```js
const favoriteFruit = Map#{
    me: "banana",
    "Alice": "kumquat",
    "Ariana": "apple",
};
const innocentNumbers = IList#[4, 8, 15, 16, 23, 42];
```

is equivalent to:

```js
const favoriteFruit = Map[Symbol.mappingLiteral]([
    [me, "banana"][Symbol.iterate](),
    ["Alice", "kumquat"][Symbol.iterate](),
    ["Ariana", "apple"][Symbol.iterate](),
][Symbol.iterate]());

const innocentNumbers = IList[Symbol.sequenceLiteral]([
    4, 8, 15, 16, 23, 42
][Symbol.iterate]());
```

This is compared to current collection construction idioms that either require `new` or ad-hoc API (e.g. static method `of`):

```js
const favoriteFruit = new Map([
    [me, "banana"],
    ["Alice", "kumquat"],
    ["Ariana", "apple"],
]);

const innocentNumbers = new IList([4, 8, 15, 16, 23, 42]);
// or
const innocentNumbers = IList.of(4, 8, 15, 16, 23, 42);
```

## Problem Statement

Current idioms for data structure literals are heavy on syntax, unsightly, and provide no parse-time safety. When constructing a `Map`, it's very easy to provide an an entry that is not a two-element `Array`. 

```js
// Error silently ignored, no ill-effect
const map = new Map([[1, "one"], [2, "two",,], [3, "three"]]);

// Error silently ignored, missing third element
const map = new Map([[1, "one"], [2, "two", 3, "three"]]);
```

This error will not generally be detected until execution time, without the assistance of static analysis tools, e.g. *ESLint*, *TypeScript*. Such a failure may occur deep into a program's execution, seconds, or days after being started.

In other languages, mapping literals either very light on syntax, provide immediate feedback at parse (or compile) time in the event of an error, or both.

In Python:

```python
{1: 'one', 2: 'two', 3: 'three'}
```

In C++17:

```c++
std::map{{1, "one"s}, {2, "two"s}, {3, "three"s}}
```

The proposed ECMAScript syntax addition would allow for equivalent terseness, and fast failure at parse time:

```js
Map#{1: "one", 2: "two", 3: "three"}
```

Existing syntax for sequences has fewer issues, but still extraneous syntax that is clumsy to type and quickly scan:

```js
new Set(["foo", "bar", "baz"])
```

The proposed ECMAScript syntax addition would allow for the following:

```js
Set#["foo", "bar", "baz"]
```

Thus far, it has been sometimes difficult in practice to encourage developers to use appropriate data types due to the relative beauty of just (ab)using `Object`, and working around the inevitable gotchas:

```js
const map = {1: "number one", "1": "string one", 2: "two", 3: "three"};
const key1 = "toString";
const key2 = 1;
console.log(map[key1]);
// [Function: toString]
console.log(map[key2]);
// "string one"

// Some concoction of Object.prototype.hasOwnProperty.call, or Object.create(null)...
```

## Proposed Solution

Abstract *sequence* syntax: *SequenceConstructor* `#[` *element* `,` *...moreElements* `]`

A sequence is constructed by calling the `Symbol.sequenceLiteral` method of *SequenceConstructor*, and passing an iterator yielding each of the specified elements.

A `Symbol.sequenceLiteral` static method would be implemented on Standard library classes `Set`, `WeakSet` and the various typed arrays.

Example:

```js
Set#["foo", "bar", "baz"]
```

Abstract *mapping* syntax: *MappingConstructor* `#{` *key* `:` *value* `,` *...moreKeyValuePairs* `}`

A mapping is constructed by calling the `Symbol.mappingLiteral` method of *MappingConstructor*, and passing an iterator yielding two-element iterators of key and value pairs.

A `Symbol.mappingLiteral` static method would be implemented on Standard library classes `Map` and `WeakMap`.

Example:

```js
Map#{1: "one", 2: "two", 3: "three"}
```

> `Object` and `Array` static methods *might* be provided for consistency, but these would be an anti-pattern due to the same performance issues as `new Object` and `new Array`.

(TODO: precedence of `#` above.)

Having two distinct protocols for sequence and mapping literals adds a little type safety, and allows a single object to be used for both, if for example a library has a natural choice for each:

```js
import * as Immutable from "Immutable";
const Imm = {
    [Symbol.sequenceLiteral]: Immutable.List[Symbol.sequenceLiteral],
    [Symbol.mappingLiteral]: Immutable.Map[Symbol.mappingLiteral],
};

const map = Imm#{1: "one", 2: "two", 3: "three"};
const list = Imm#[1, 2, 3];
const set = Immutable.Set#[1, 2, 3];
```

## FAQ

### Why introduce new syntax?

ECMAScript is occasionally overlooked by the wider programming community for a variety of reasons. Introducing syntax that *promotes* the use of correct (or just more convenient) data types may increase the perceived suitability of the language, and will reduce bugs.


### Why no barewords?

Barewords *do* make sense for `Object` literals and and property access, because keys are generally string type (symbols notwithstanding). Consider:

```js
const obj = {
    foo: 42,
};

// Read
obj.foo

// Write
obj.foo = 43;
```

However, for more general mapping types like `Map`, `Immutable.Map`, etc., literal string keys must be spelled as actual string literals, at both construction *and access*:

```js
const map = new Map([
    ["foo", 42],
]);

// Read
map.get("foo")

// Write
map.set("foo", 43);
```

Furthermore, a large utility of bona fide mapping types is the fact that keys may generally be of any type. Having to surround non-string keys with tokens like `[]` would be inconvenient in practice, and somewhat visually ambiguous about whether the key is actually an `Array` object.

```js
// NOT THE PROPOSED BEHAVIOUR:
IMap#{
    "1": "string 1",
    1: "string 1, again...",
    [1]: "number 1",
    [[1]]: "array of number 1",
    [1, 2]: "number 2???",  // Comma operator? Not obvious if valid.
    [[1, 2]]: "array of numbers 1 and 2",
    [IList#[1]]: "immutable list of number 1",
};

// Proposed behaviour. Keys are exactly as they would appear anywhere else.
IMap#{
    "1": "string 1",
    1: "number 1",
    [1]: "array of number 1",
    [1, 2]: "array of numbers 1 and 2",
    [[1, 2]]: "array of (array of numbers 1 and 2)",
    IList#[1]: "immutable list of number 1",
}
```

Still furthermore, many map use cases don't involve string keys at all. Having to wrap *every* key in `[]` would be fairly annoying to type, and a constant source of errors if people forget.

```js
// Ambiguous?
IMap#{
    [1]: "one",
    [2]: "two",
    [3]: "three",
    [4]: "four",
    [5]: "five",
}
```

Similarity to `Object` literal syntax here would, at best, be incomplete, due to object shorthand syntax:

```js
// What does this mean?
Map#{
    get foo() { return 100; },
    set foo(v) {}
    constructor() {}
}
```

### Why iterators?

Iterators are a more fundamental abstraction than `Array`s. This avoids having to necessarily have an *N*-element `Array` coexist with an *N*-element collection during construction.

It also avoids placing any additional special burden on the `Array` type (it is already used for rest arguments, whereas argument spreading works in terms of arbitrary literals.)


### Why extensible?

Making this syntax extensible and not coupling it to any of the built-in collection types allows for more natural, first-class use of domain-specific, or frankly more modern data structures. This effectively *future-proofs* the syntax for any new library types that may be added to the standard in the future, e.g. a keys-as-values `Dict` type.


## Possible Variations

### Alternative mapping tokens

If the similarities of the `{key: value}` syntax for mapping literals bears too much resemblance to `Object` literals to excuse the differences (e.g. barewords), then other alternatives could be reasonably considered, e.g. `{key -> value, ...}`

Note that alternatives such as `[key -> value, ...]` would be ambiguous in the zero element case, i.e. is `Constructor#[]` a mapping literal or a sequence literal?


### Alternatives to the `#` token

The `#` token is frequently chosen for proposed syntax additions. It goes without saying that any token could be used for this proposal with little or no impact on its value. It could be argued that a "hash" is, at a stretch, a mnemonic for a mapping type, but this clearly doesn't apply to sequences.


## References

  1. *Map Literal* discussion on es-discuss: [https://esdiscuss.org/topic/map-literal](https://esdiscuss.org/topic/map-literal)
