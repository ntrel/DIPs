# Missing Elements in Static Array Initializers

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | Nick Treleaven ([@ntrel](https://github.com/ntrel))             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract
Deprecate missing elements in statically initialized static arrays in the next edition,
unless the missing elements in the array initializer are opted-into using new syntax.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Alternatives](#alternatives)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [History](#history)

## Rationale
The dmd front-end is inconsistent when a static array declaration of type `E[n]` has an
array initializer:

- Static initialization allows fewer elements than the static array length. The
  remaining elements are set to `E.init`.
- Dynamic initialization with fewer elements than the static array length
  is a compile-time error.

```d
int[5] x = [1, 2, 3, 4]; // Allowed

void f() {
	int[5] y = [1, 2, 3, 4]; // Error, too few elements
}
```

When there are many elements in the initializer, it is easy to miscount and accidentally
miss out elements that should not be default initialized. To avoid this, and to make the
two cases consistent, the static initialization case should be deprecated.

Note that a static declaration with an expression
[initializer](https://dlang.org/spec/declaration.html#initializers) is rejected when the
expression has a shorter length:

```d
int[5] x = [1, 2, 3, 4]; // Allowed
int[5] y = [1, 2, 3, 4] ~ []; // Error, too few elements
```

That is another inconsistency which would be fixed by deprecating missing elements in an
array initializer.

### Bug Detection
[Dennis:](https://github.com/dlang/dmd/issues/21817#issuecomment-3271935565)

> In dmd, druntime and Phobos, there has been 1 legit case and 4 bugs uncovered [by
> proposed deprecation]. Small sample size, but 80% erroneous usage is clearly a problem

[Manu:](https://github.com/dlang/dmd/issues/21817#issuecomment-3272787021)

> My own data found 3 more, so that's 4 errors in about 8 instances of this pattern; ~50% error rate!

## Prior Work

### C
https://en.cppreference.com/w/c/language/array_initialization.html

> All array elements that are not initialized explicitly are empty-initialized.

```c
int x[3] = {1, 2}; // Allowed
```

### C++
Fixed size array [library type](https://en.cppreference.com/w/cpp/container/array.html).

```c++
#include <array>

std::array<int, 3> a = {1, 2}; // Allowed
```
This is perhaps allowed due to `std::array` being a replacement for a C array.

### Rust
Rust [fixed size arrays](https://doc.rust-lang.org/reference/types/array.html).

```rust
const array: [i32; 3] = [1, 2]; // Error: expected an array with a size of 3, found one with a size of 2
```

### Haskell
Haskell [fixed size arrays](https://www.haskell.org/onlinereport/haskell2010/haskellch14.html).
The following declares a fixed size array with indexes 0, 1 and 2 and initializes it
with only two elements:

```hs
main = do
    let myArray = listArray (0, 2) [1, 2] -- Error: undefined array element
```

### Go
Go [fixed size arrays](https://go.dev/ref/spec#Composite_literals) - see under *Array and slice literals*.

> If fewer elements than the length are provided in the literal, the missing elements are set to the zero value for the array element type

```go
var myArray [3]int = [3]int{1, 2} // Allowed
```

### Require an Index
Dennis [has a PR](https://github.com/dlang/dmd/pull/21821) which deprecates missing
elements, requiring an index for one of the elements in the initializer. See
[Alternatives](#alternatives) for a critique.

## Description
Static initialization from an array initializer with missing elements will be deprecated
in the next edition, except when:

1. new `...` syntax is used
1. an element initializer has an index specified
1. the array initializer is empty `[]`

The second case is already allowed to have missing elements by design (including
for dynamic initialization). The third case is not considered bug-prone.

Examples:

```d
int[3] v = [1, 2]; // Deprecated
int[3] w = [1: 4]; // OK, element index given
int[3] x = [];     // OK, empty

void main() {
    assert(w == [0, 4, 0]);
    assert(x == [0, 0, 0]);
}
```

### New Syntax
[*ArrayInitializer*](https://dlang.org/spec/arrays.html#array-initializers) grammar will
be extended:

```diff
 ArrayInitializer:
-    [ ArrayElementInitializers ]
+    [ ArrayElementInitializers ,opt ]
+    [ ArrayElementInitializers , ... ]

 ArrayElementInitializers:
     ArrayElementInitializer
-    ArrayElementInitializer ,
     ArrayElementInitializer , ArrayElementInitializers
```

A declaration of type `E[n]` with an array initializer `[elements, ...]` will have each
missing element initialized by `E.init`. This applies to both static and dynamic
initialization. A nested array initializer can also use this syntax when it relates to
a static array.

Examples:

```d
int[3] x = [1, 2, ...];
int[3][] slice = [[1, ...], [4, 5, ...]]; // nested array initializers

void main() {
    assert(x == [1, 2, 0]);
    assert(slice == [[1, 0, 0], [4, 5, 0]]);
}
void f(int i) {
    int[3] y = [i, ...];
    assert(y == [i, 0, 0]);
}
```

It is an error to use `[elements, ...]` initializer syntax when:

- the declaration is not a static array
- there are no missing elements
- an initializer element has an index specified e.g. `2: expr`

The new syntax can be supported in the default edition too, as it does not break anything.

## Alternatives

### Manually adding missing elements instead of `[elements, ...]`
This would be tedious when there are a lot of missing elements. It would likely need
an additional (global) declaration when there is no short token available to use for
`E.init` (e.g. when E is a struct template instantiation).

### Using `[0: first, tail]` instead of `[first, tail, ...]`
This is not as intuitive to read, but could be made to work. The compiler could suggest
it when issuing a missing elements error. People new to the syntax would likely ask about
it on the forum. Note: Currently using a leading `0:` still errors for dynamic
initialization.

### Adding missing elements with a value sequence template
```d
import core.foo : repeat;
int[100] x = [1, 2, 3, repeat!(0, 97)];
```

Drawbacks: Needs specifying the number of missing elements and adjusting when
adding/removing elements. May cause template bloat. Needs an import somewhere.

### Using slice assignment instead of `[elements, ...]`
```d
void f() {
    int[5] x;
    x[0..4] = [1, 2, 3, 4];
}
```

Drawbacks: Needs to be done in a module constructor for global initialization, or with
CTFE for static initialization. The number of elements on the right hand side needs to be
provided for the slice, which could be hard to count. Either of these would make
upgrading existing code more awkward.

## Breaking Changes and Deprecations
A deprecation for the next edition is chosen so that:

- default edition users do not get deprecation messages when simply upgrading a single
  compiler version
- when upgrading to the next edition, users have some time to upgrade their code instead of
  being required to do so (by an error) before they can use the next edition

A tool should be provided to automatically update declarations that would be deprecated,
e.g. [`dscanner fix --applySingle`](https://github.com/dlang-community/D-Scanner?tab=readme-ov-file#auto-fixing-issues).

### Compiler Suggestions
When issuing a deprecation for missing elements, the compiler should suggest using
`[elements, ...]` initializer syntax.
When such an array declaration (of type `E[n]`) has a nonzero `E.init`, the compiler
could show a supplemental message to remind the user to take care if they are porting
code from C. This would help avoid bugs with arrays which should have missing elements
zeroed (which could be done at runtime).

## Reference
- Github issue: https://github.com/dlang/dmd/issues/21817.
- Walter's `[elements, ...]` [implementation](https://github.com/WalterBright/dmd/commit/cbcd47975fec7af7467c12fac83ef13a05dad745).
- [DLF September 2025 Monthly Meeting](https://forum.dlang.org/post/ucuhbblifjcjkfvikbqm@forum.dlang.org) -
see part of the *Static array length inference* discussion.

## Copyright & License
Copyright (c) 2026 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.
