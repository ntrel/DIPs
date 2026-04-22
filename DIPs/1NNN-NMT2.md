# Call-site enforcement for `@nogc`/`@safe` functions with `scope` delegate parameters

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | Nick Treleaven ([@ntrel](https://github.com/ntrel))             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract
Defer `@nogc` and `@safe` attribute checks for a function with a `scope` delegate
parameter to the caller of the function (if needed).

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [History](#history)

## Rationale
In a function `f`, in order to call a delegate parameter, the delegate has to match
the `@nogc` and `@safe` attributes which `f` is declared with.
This means a caller of `f` is restricted to passing a delegate
argument which matches `f`'s attributes, even if the caller does not need to comply
with those attributes, and the parameter does not escape `f`.

If `f` doesn't support
those attributes, functions that are required to support them cannot call `f`.
To support both cases fully, `f` would need overloads for each of the 4 possible
attribute combinations (none, one of each, and both). That is not practical.

```
void foo(scope void delegate() @nogc @safe sd) @nogc @safe
{
    sd();
}

void delegate() @safe d1;
void delegate() @system d2;

void bar() @safe {
    foo(d1); // OK
    foo(d2); // Error: `d2` is not @safe - necessary
}

void bar() @nogc @system {
    foo(d1); // OK
    foo(d2); // Error: `d2` is not @safe - not useful
}
```
The error about needing the delegate to comply with `@safe` when it is being called from
a `@system` function is not useful, because `bar` is allowed to execute unsafe operations.

Another problem is that (when applicable),
[attribute inference](https://dlang.org/spec/function.html#function-attribute-inference)
for `f` itself is effectively disabled when a delegate parameter does not
specify attributes.

## Analysis

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
- [This forum post](https://forum.dlang.org/post/scpdxcikxgaywgcvknok@forum.dlang.org>)
  led to <https://github.com/dlang/DIPs/blob/master/DIPs/other/DIP1041.md> -
  This DIP further develops that idea.

## Description
1. Allow a function `f` with `@nogc` and/or `@safe` attributes and a
   `scope` delegate parameter to treat the delegate as if it had matching `@nogc`
   and/or `@safe` attributes when performing semantic analysis of `f`.
2. If a caller of `f` has a `@nogc` or `@safe` attribute matching `f`'s attributes, the delegate
   passed to `f` must comply with the matching attributes.

```d
void noAttributes();
void delegate() @safe safeDel;

void foo(scope void delegate() sd, void delegate() d) @safe {
    sd(); // OK, `sd` isn't checked for @safe here
    d(); // Error: delegate parameter `d` is not `@safe` or `scope`
    noAttributes(); // Error: can't call `@system` function
    sd = { noAttributes(); }; // Error: cannot assign `@system` delegate to `sd`
    sd = safeDel; // OK
    sd(); // still OK
}
void bar() @safe {
    foo(safeDel); // OK, `foo` is marked @safe
    foo({ noAttributes(); }); // Error: can't pass `@system` delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, `baz` is @system
}
```
3. When [inferring attributes](https://dlang.org/spec/function.html#function-attribute-inference)
   for a function `f` with a `scope` delegate parameter, additional analysis is undertaken.
   If the parameter is not reassigned and its address is not used as a mutable value,
   the parameter will be treated as if it was marked `@nogc @safe` for the purposes
   of inferring attributes for `f`.
   Note that the parameter itself *is not inferred* with those attributes.
   Attribute inference then proceeds normally, so that if the body of `f` complies
   with `@nogc` and/or `@safe` then `f` is inferred with those attributes.

```d
void noAttributes();
void delegate() @safe safeDel;

// `foo` is inferred @safe
auto foo(scope void delegate() sd) {
    sd(); // OK, `sd` isn't checked for @safe here
    return new int;
}
void bar() @safe {
    foo(safeDel); // OK, call is @safe
    foo({ noAttributes(); }); // Error: can't pass `@system` delegate literal
}
void baz() @system {
    foo({ noAttributes(); }); // OK, `baz` is @system
}
```
First, the compiler attempts to infer the attributes of `foo`. Delegate parameter `sd`
is `scope`, so whilst analysing each statement in `foo`, it determines that `sd`
is called, but it is not assigned to and its address is not taken.
Therefore, `sd` can be treated as `@nogc @safe` (at this stage).

As there was no `@system` operation found, `foo` will be inferred as `@safe`.
Due to the *NewExpression*, `foo` will not be inferred as `@nogc`.
Note that `sd` itself is not inferred with any attributes.
Then:
- `foo` compiles due to DIP point 1.
- The call `foo(safeDel)` in `bar` then compiles due to DIP point 2.
- The call `foo({ noAttributes(); })` in `baz` compiles because `foo`'s delegate
  parameter does not require `@safe`, and `baz` is `@system`.

4. The above DIP points can also apply for delegate parameters which are
   [inferred as `scope`](https://dlang.org/spec/function.html#function-attribute-inference).
5. The above DIP points also apply equivalently for function pointers instead of delegates.

## Breaking Changes and Deprecations
This section is not required if no breaking changes or deprecations are anticipated.

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.

## Reference
- <https://github.com/dlang/dmd/discussions/22905> - a different proposal to solve the
  same problem.

Optional links to reference material such as existing discussions, research papers
or any other supplementary materials.

## Copyright & License
Copyright (c) 2024 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.
