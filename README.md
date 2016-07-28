Shallow Object Equality Test for ECMAScript
-------------------------------------------

TL;DR: `Object.shallowEqual(a, b)` does a _memcmp_ on the internal object representation and returns `true` if it is equal. The exact semantics of the object representation are undefined so a valid implementation may always return `false`.

## The Problem

Programming paradigms that rely on immutable data structures use memoization as an optimization technique of pure functions. Libraries like React and others currently rely heavily on doing object comparisons to know if a calculation can bail out. A memoized result can then be reused.

#### Memoization

```js
function eq(a, b) {
  return Object.is(a, b);
}

function memoize(fn) {
  let lastArg, lastResult;
  return function(arg) {
    if (lastArg !== undefined && eq(lastArg, arg)) {
      return lastResult;
    }
    lastArg = arg;
    lastResult = fn(arg);
    return lastResult;
  };
}

function calc(obj) {
  return obj.x + obj.y;
}

let memoizedCalc = memoize(calc);
```

```js
let obj = { x: 1, y: 2 };

let res1 = memoizedCalc(obj); // slow

let res2 = memoizedCalc(obj); // quick

let res3 = memoizedCalc({ x: 3, y: 4 }); // slow
```

The problem with reference equality tests is that often objects are recreated with the same nested data:

```js
function transform([x, y]) {
  return { x: x / 2, y: y / 2 };
}

let arr = [2, 4];

let res1 = memoizedCalc(transform(arr)); // slow

let res2 = memoizedCalc(transform(arr)); // slow
```

This is a very common pattern in React.

#### Shallow Equality

To avoid this problem, libraries implement shallow comparisons of object by comparing the values one level deep:

```js
function eq(a, b) {
  if (Object.is(a, b)) {
    return true;
  }

  if (typeof a !== 'object' || a === null ||
      typeof b !== 'object' || b === null) {
    return false;
  }

  if (Object.getPrototypeOf(a) !== Object.getPrototypeOf(b)) {
    return false;
  }

  const keysA = Object.keys(a);
  const keysB = Object.keys(b);

  if (keysA.length !== keysB.length) {
    return false;
  }

  for (let i = 0; i < keysA.length; i++) {
    if (
      !Object.prototype.hasOwnProperty.call(b, keysA[i]) ||
      !Object.is(a[keysA[i]], b[keysA[i]])
    ) {
      return false;
    }
  }

  return true;
}
```

You can create an optimized path for specific known object signatures if you know them upfront but these generic variants are not optimized by VMs and requires lots of introspection into the internal hidden class representations to look up the keys in the respective class.

Meanwhile, on native architectures the equivalent operation can be as little as a few CPU instructions depending on architecture and optimizations.

#### Deep Equality

This proposal doesn't provide a complete solution for deep equality. However, any implementation of strict deep equality in user space currently suffers when the values of an object are equal because there is no fast way to bail out.

Such implementations can benefit from a fast path for the leaf object case or when part of a deep data structure was reused.

## Implementation

It is expected that VMs have an implementation that roughly models a memory layout like this:

```
[
  hidden class pointer,
  maybe a prototype pointer,
  expando pointer,
  numeric fields pointer,
  inline slots...
]
```

In the example above, this object representation can be efficiently compared with a `memcmp`:

```c
bool eq(JSObject *a, JSObject *b) {
  if (a->hiddenClass !== b->hiddenClass) {
    return false;
  }
  size_t instanceSize = a->hiddenClass->instanceSize;
  return !memcmp(a, b, instanceSize);
}
```

#### Caveats

The JavaScript object model isn't as simple as a C struct though. The above implementation isn't strictly equivalent to what the user space shallow comparison does. Specifically it is more restrictive in what it returns `true` for.

Strings in JavaScript VMs are represented by pointers to other string segments or ropes. That means that two different string pointers that are semantically equal, may be represented by two different pointers and would return `false` for a comparison.

Similarly some ranges of floating point numbers are represented as boxed values. That means that the same semantically equivalent number may be represented by two different pointers.

Not all fields are inline slots in the same allocation as the object. Sometimes expando properties and numeric fields are allocated separately. This means that a simple memcmp wouldn't include those fields.

The enumeration order of fields, as well as their configuration, are often stored separately from the object itself. Two different configurations would have different pointers. There may also be two objects which are semantically equivalent but were constructed in different ways so they don't share the same configuration object. They would also return false.

However, despite these caveats, this heuristic is still useful because most of the time it will be enough. If the comparison returns `false`, a library can choose to keep going deeper or simply perform the calculation again. It is not uncommon to have a large object where performing the next piece of calculation is faster than performing a more expensive comparison.

For these reasons, this proposal will not specify a strict requirement for when an implementation have to return `true`. It is fully compliant to always return `false`. It is OK for an implementation to do a more expensive comparison to be able to return `true` in more cases, but those semantics are strictly defined.

Function objects also contains pointers to source and closures. Two functions with the same source code and same closure scope maybe considered equivalent, even though it is not currently observable. This specification will allow an implementation to return `true` in this case but won't be required to do so.

## Specification

This will specify a new method called `Object.shallowEqual` which takes two arguments (`x` and `y`). An implementation may always return `false` from this function. It is never required to return `true`.

If the type of the two arguments are not the same then this function returns `false`.

An implementation may only return `true` if the following conditions are satisfied per type:

- Undefined: SameValue(x, y)
- Null: SameValue(x, y)
- Boolean: SameValue(x, y)
- Number: SameValue(x, y)
- String: SameValue(x, y) (It is recommended that string ropes only do a shallow comparison on internal pointers rather than deep comparison.)
- Symbol: SameValue(x, y)
- Host object (provided by the JS environment): Implementation-dependent
- Object:
  If the `[[Prototype]]` of each object is the same Object value, all the own properties have all the same property names and the `[[Value]]`, `[[Get]]`, `[[Set]]`, `[[Writable]]`, `[[Enumerable]]` and `[[Configurable]]` internal slots of the properties all have the same value according to the semantics defined by SameValue. The enumeration order of properties does not have to be the same.
- Function:
  The same rules as for Object applies. Additionally, the two functions have to have the same values for all its internal slots as defined by [9.2](https://tc39.github.io/ecma262/#sec-ecmascript-function-objects). These currently include: `[[Environment]]`, `[[FormalParameters]]`, `[[FunctionKind]]`, `[[ECMAScriptCode]]`, `[[ConstructorKind]]`, `[[Realm]]`, `[[ScriptOrModule]]`, `[[ThisMode]]`, `[[Strict]]`, `[[HomeObject]]`. (TODO: It is plausible that this could be relaxed to include other equivalent functions if VMs start optimizing such shared functions in a way that it is not safe to know if one of these slots are actually the same.)

## Risks of Undefined Behavior

The biggest risk involved with this proposal is that code on the web starts relying on a particular implementation of the data structure and comparison. E.g.

```js
let a = { 0: 1, x: 2 };
a.y = 3;
let b = { 0: 1, x: 2 };
b.y = 3;
if (Object.shallowEqual(a, b)) {
  throw new Error();
}
```

A current VM might treat numeric properties and expandos as a separate data structure. However, if that changes the `shallowEqual` call may start returning `true` since they are equivalent. Which would then break that code. Code can also rely on the opposite being true.

To mitigate this risk, we suggest that browsers use a technique to minimize reliance on undefined behavior. Such as always returning `false`, or always do a deeper comparison, for a certain cohort of users.

## Possible Issues

It is currently unknown how this would work with flattened prototype chains. E.g. is is possible to know the object reference identity of the prototype if the fields are flattened into the object slots? More research is needed on the implementation details of such experimental work.

The name `Object.shallowEqual` might be too prominent when this is really a power-user feature. It might be confused with something like the user space implementation that gives stronger guarantees.

## Security Considerations

This feature exposes a few new capabilities. It is not possible to tell a string made up of ropes from one that is not. It is possible to tell if a number is boxed or not, which might give some clues to the architecture that you're running on. Comparing functions may also expose whether its scope is shared or not.

This opens up a potential communication channel. However, it is our belief that this is not opening up any new exploitable surfaces that are not already exploitable. E.g. through timing attacks.

If there are concerns, a more secure environment such as CSP or SES may choose to always return `false`.

## Why not a built-in memoization feature?

Some languages have the ability to memoize a pure function as a built-in feature. That would require specification of what a pure function really is, which is a big ask for JavaScript. Especially considering that there are no refentially transparent data structures such as rich value types and every useful operation is on a mutable prototype.

It is much easier to leave that to user space where the rule can be much more loose.

Another problem with that approach is that current frameworks rely on these tests to perform side-effects conditionally. Therefore you have to know if the comparison failed or not. For example, React does mutations on the DOM but _only_ if the return value did change. Therefore the underlying capability needs to be exposed anyway.

## [Status of this Proposal](https://github.com/tc39/ecma262)

This has not yet been presented to the TC39 commitee. It is my intention to propose this as a Stage 0 proposal to ECMAScript. However, before we do that we would like to run some experiments in browsers to get some numbers how this could affect performance.
