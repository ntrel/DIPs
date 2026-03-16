# Missing Elements in Static Array Initializers

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | Nick Treleaven ([@ntrel](https://github.com/ntrel))             |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract
Deprecate missing elements in statically initialized static arrays in the next edition,
unless the missing elements in the array initializer are marked using new syntax.

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

### Rust
Rust has [fixed size array types](https://doc.rust-lang.org/reference/types/array.html).

```rust
const array: [i32; 3] = [1, 2]; // Error: expected an array with a size of 3
```

### Require an Index
Dennis [has a PR](https://github.com/dlang/dmd/pull/21821) which deprecates missing
elements, requiring an index. See [Alternatives](#alternatives) for a critique.

## Description
Static initialization from an array initializer with missing elements will be deprecated
in the next edition, except when:

- new `...` syntax is used
- an element initializer has an index specified
- the array initializer is empty `[]`

Note: The second case is already allowed to have missing elements by design (including
for dynamic initialization).

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
+    [ ArrayElementInitializers , ... = Initializer ]

 ArrayElementInitializers:
     ArrayElementInitializer
-    ArrayElementInitializer ,
     ArrayElementInitializer , ArrayElementInitializers
```

A declaration of type `E[n]` with an array initializer `[elements, ...]` will have each
missing element initialized by `E.init`.

The [*Initializer*](https://dlang.org/spec/declaration.html#initializers) form is used to
initialize each of the missing elements in the array initializer. It can be:
- an expression
- an array initializer (when `E` is an array type)
- a struct initializer
- `void` - an implementation would leave just the missing elements uninitialized

All expressions in *Initializer* must be known at compile-time, even for dynamic
initialization. This makes it illegal for an expression to have side-effects, so each
missing element has the same value.

The *Initializer* form is useful when porting code from C which needs missing elements to
be zeroed. This also complements D's special handling of character array initialization
[from a string literal](https://dlang.org/spec/arrays.html#static-string) where missing
elements are zeroed.

Examples:

```d
int[3] x = [1, 2, ...];
int[3] y = [1, ... = 5];
float[3] z = [1.5F, ... = 0F]; // zero, not float.init
int[3][] slice = [[1, ...], [4, ... = 5]]; // nested static array

void main() {
    assert(x == [1, 2, 0]);
    assert(y == [1, 5, 5]);
    assert(z == [1.5F, 0F, 0F]);
    assert(slice == [[1, 0, 0], [4, 5, 5]]);

    int[3] a = [1, ... = x[2]]; // Error, `x` is not known at compile-time
    int[3] b = [1, 2, ... = void];
    assert(b[0..2] == [1, 2]);
    // b[2] is unknown
}
```

It is an error to use `[elements, ...]` initializer syntax when:

- the declaration is not a static array
- there are no missing elements
- an initializer element has an index specified e.g. `2: expr`

However, `[elements, ... = init]` could be supported when there is an element with an
index specified and there is a missing element. In that case, every missing element must
be initialized with `init`, not just trailing elements. Otherwise it should be an error.

The new syntax can be supported in the default edition too, as it does not break anything.

## Alternatives

### Manually adding missing elements instead of `[elements, ...]`
This would be tedious when there are a lot of missing elements. It would likely need
an additional (global) declaration when there is no short token available to use for
`E.init`.

### Using `[0: first, tail]` instead of `[first, tail, ...]`
This is not as intuitive to read, but could be made to work. The compiler could suggest
it when issuing a missing elements error. People new to the syntax would likely ask about
it on the forum. Note: Currently using a leading `0:` still errors for dynamic
initialization.

If there was no `...` syntax then the `... = init` syntax would be more of a
special case, and not having that has its own drawbacks (see below).

### Using an enum instead of `[elements, ...]`
```d
enum xdata = [1, 2, 3, 4];
immutable int[xdata.length] x = xdata; //or slice
```

Drawbacks: This would be awkward to manually upgrade code, and adds a (global) declaration
each time which is only used once. It is also awkward to type and users would have to be
taught to use it.

### Adding missing elements with a value sequence template
```d
import core.foo : repeat;
int[100] x = [1, 2, 3, repeat!(0, 97)];
```

Drawbacks: Needs specifying number of missing elements and adjusting when adding/removing
elements. May cause template bloat. Needs an import somewhere.

### Using slice assignment instead of `[elements, ... = init]`
```d
void f() {
    int[5] x;
    x[0..4] = [1, 2, 3, 4];
}
```

Drawbacks: Needs to be done in a module constructor for global initialization, or with CTFE
for immutable/static initialization. This would make upgrading existing code more awkward.

### Not supporting `... = init`
Drawbacks similar to above, and it would make porting C code harder which sometimes needs
missing elements to be zeroed.

## Breaking Changes and Deprecations
A deprecation for the next edition is chosen so that:

- default edition users do not get deprecation messages when simply upgrading a single
  compiler version
- when upgrading to the next edition, users have some time to upgrade their code instead of
  being required to do so before they can use the next edition

A tool should be provided to automatically update declarations that would be deprecated,
e.g. [`dscanner --applySingle`](https://github.com/dlang-community/D-Scanner?tab=readme-ov-file#auto-fixing-issues).

### Missing Elements Compiler Suggestions
When issuing a deprecation for missing elements, the compiler should suggest using
`[elements, ...]` initializer syntax.

When the array declaration of type `E[n]` has a nonzero `E.init` and `0` is a valid
element initializer, the compiler should also remind the user to use
`[elements, ... = 0]` initializer syntax if they are porting code from C. This can help
avoid bugs with character arrays that should be zero-terminated, or floating point arrays
which should have missing elements zeroed.

## Reference
- Github issue: https://github.com/dlang/dmd/issues/21817.
- [DLF September 2025 Monthly Meeting](https://forum.dlang.org/post/ucuhbblifjcjkfvikbqm@forum.dlang.org)
- Walter's `[elements, ...]` [implementation](https://github.com/WalterBright/dmd/commit/cbcd47975fec7af7467c12fac83ef13a05dad745).

## Copyright & License
Copyright (c) 2026 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.
