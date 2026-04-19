# Call-site enforcement for `@nogc`/`@safe` functions with `scope const` delegate parameters

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | Nick Treleaven ([@ntrel](https://github.com/ntrel))             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract
Defer `@nogc` and `@safe` attribute checks for a non-mutable `scope` delegate parameter
to the caller of the function (if needed).

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [History](#history)

## Rationale
TODO

- If a function `f` has a delegate parameter marked `scope`, it must not escape `f`.
  Therefore it is reasonable to assume it will be called by `f`, in order for the delegate
  to be useful.
- If the delegate is not mutable, it cannot be reassigned to a delegate which doesn't
  comply with `f`'s attributes.
- When `f` has `@nogc` or `@safe` attributes, they only need to be enforced for the
  delegate argument when `f` is called by a `@nogc` or `@safe` function.
  If a caller of `f` does not have those attributes,
  `f`'s delegate argument body could freely allocate on the GC or use `@system` operations. In
  addition, both caller scenarios are supported with the same machine code of `f`. In contrast,
  `nothrow` and `pure` would make code generation problematic if this DIP
  treated them like `@nogc` and `@safe`.

## Prior Work
- <https://forum.dlang.org/post/scpdxcikxgaywgcvknok@forum.dlang.org> by Quirin Schroll -
  see solution. This DIP strengthens that idea by adding some restrictions.
- <https://github.com/dlang/dmd/discussions/22905> - a different proposal to solve the
  same problem.

## Description
1. Allow a function `f` with a `@nogc` or `@safe` attribute and a
   non-mutable `scope` delegate parameter to call the delegate without checking the delegate has
   compatible attributes. Any other expression `f` evaluates must still comply with
   `f`'s attributes.
2. If a caller of `f` has a `@nogc` or `@safe` attribute matching `f`'s attributes, the delegate
   passed to `f` must comply with the matching attributes.

Note: Examples require [`-preview=in`](https://dlang.org/spec/function.html#in-params) for
the `in` parameter storage class (to mean `scope const`).
```d
void noAttributes();
void delegate() @nogc nogcDel;

void foo(in void delegate() id, void delegate() d) @nogc {
    id(); // OK, `id` isn't checked for @nogc here
    d(); // Error, `d` is not marked @nogc or `in`
    noAttributes(); // Error, can't call non-@nogc function
}
void bar() @nogc {
    foo(nogcDel); // OK, `nogcDel` is @nogc and `foo` is marked @nogc
    foo({ noAttributes(); }); // Error, can't call non-@nogc delegate literal
}
void baz() {
    foo({ noAttributes(); }); // OK, `baz` isn't @nogc
}
```
```d
void noAttributes();
void delegate() @safe safeDel;

void foo(in void delegate() id) @safe {
    id(); // OK, `id` isn't checked for @safe here
    noAttributes(); // Error, can't call @system function
}
void bar() @safe {
    foo(safeDel); // OK, `safeDel` is safe and `foo` is marked safe
    foo({ noAttributes(); }); // Error, can't call @system delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, `baz` is @system
}
```
3. When [inferring attributes](https://dlang.org/spec/function.html#function-attribute-inference)
   for a function `f` with a non-mutable `scope` delegate parameter, the delegate parameter
   will be treated as if it were marked `@nogc @safe`.

```d
void noAttributes();
void delegate safeDel() @safe;

// template `foo` is not marked @safe
void foo()(scope const void delegate() cd) {
    cd(); // OK, `cd` isn't checked for @safe here
}
void bar() @safe {
    foo(safeDel); // OK, `safeDel` is safe and `foo` call is inferred safe
    foo({ noAttributes(); }); // Error, can't call @system delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, `baz` is @system
}
```
4. The above points also apply for a non-mutable `scope` *function pointer* parameter.
5. The above points can also apply for delegate/function pointer parameters which are
   [inferred as `scope`](https://dlang.org/spec/function.html#function-attribute-inference).
   `scope` parameter inference must be done before `f`'s attribute inference happens.

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
