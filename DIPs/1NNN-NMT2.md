# Deferred attribute enforcement for functions with `scope const` delegate parameters

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | Nick Treleaven ([@ntrel](https://github.com/ntrel))             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract
Defer attribute checks for `scope const` delegate parameters to the caller of the function,
rather than the function itself.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [History](#history)

## Rationale
- If a function `f` has a delegate parameter marked `scope`, it must not escape `f`.
  Therefore it is reasonable to assume it will be called by `f`, in order for the delegate
  to be useful.
- If the delegate is not mutable, it cannot be reassigned to a delegate which doesn't
  comply with `f`'s attributes.

TODO

## Prior Work
- <https://forum.dlang.org/post/scpdxcikxgaywgcvknok@forum.dlang.org> by Quirin Schroll -
  see solution. This DIP builds on that idea by adding `scope const` restrictions to the
  parameter.
- <https://github.com/dlang/dmd/discussions/22905>

## Description
1. Allow a function `f` with a `nothrow`/`pure`/`@nogc`/`@safe` attribute and a
   `scope const` delegate parameter to call the delegate without checking the delegate has
   compatible attributes. Any other expression `f` evaluates must still comply with
   `f`'s attributes.
2. If a caller of `f` has an attribute matching `f`'s attributes, the delegate
   passed to `f` must comply with the matching attributes.
3. For optimization and code generation, `f` cannot be treated as truly `pure` or `nothrow`
   unless all of its `scope const` delegate parameters are also marked `pure`/`nothrow`.
4. The same applies for a `scope const` *function pointer* parameter, though that is omitted
   throughout this DIP.
5. The same applies for `scope immutable` delegate/function pointer parameters.
6. The same applies for `const`/`immutable` delegate/function pointer parameters which are
   [inferred as `scope`](https://dlang.org/spec/function.html#function-attribute-inference).

Note: Examples require [`-preview=in`](https://dlang.org/spec/function.html#in-params) for
the `in` parameter storage class (to mean `scope const`).
```d
void noAttributes();
void delegate() pure pureGD;

void foo(in void delegate() id, void delegate() d) pure {
    id(); // OK, id's purity isn't checked here
    d(); // Error, d is not pure
    noAttributes(); // Error, can't call impure function
}
void bar() pure {
    foo(pureGD); // OK, pureGD is pure and foo is marked pure
    foo({ noAttributes(); }); // Error, can't call impure delegate literal
}
void baz() {
    foo({ noAttributes(); }); // OK, baz isn't pure
}
```
```d
void noAttributes();
void delegate() @safe safeGD;

void foo(in void delegate() id) @safe {
    id(); // OK, id's safety isn't checked here
    noAttributes(); // Error, can't call @system function
}
void bar() @safe {
    foo(safeGD); // OK, safeGD is safe and foo is marked safe
    foo({ noAttributes(); }); // Error, can't call @system delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, baz is @system
}
```
7. When [inferring attributes](https://dlang.org/spec/function.html#function-attribute-inference)
   for a function `f` with a `scope const` delegate parameter, the delegate parameter will be
   treated as if it were marked `nothrow pure @nogc @safe`.
8. The same applies when a `const` delegate parameter is inferred as `scope`.

```d
void noAttributes();
void delegate safeGD() @safe;

// template foo is not marked @safe
void foo()(const void delegate() cd) { // cd is inferred as scope
    cd(); // OK, cd's safety isn't checked here
}
void bar() @safe {
    foo(safeGD); // OK, safeGD is safe and foo call is inferred safe
    foo({ noAttributes(); }); // Error, can't call @system delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, baz is @system
}
```

## Breaking Changes and Deprecations
This section is not required if no breaking changes or deprecations are anticipated.

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.

## Reference
Optional links to reference material such as existing discussions, research papers
or any other supplementary materials.

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.
