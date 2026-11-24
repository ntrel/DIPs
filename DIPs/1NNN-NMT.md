# Tuple Unpacking Statements

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                          |
| Author:         | (your name and contact email address)                           |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Draft                                                           |

## Abstract

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [History](#history)

## Rationale
This follows on from [DIP1053](https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1053.md).
Unpacking assignments [were postponed](https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1053.md#postponed-unpacking-assignments)
from the original DIP. This was to avoid repurposing the comma expression syntax (and to
simplify the DIP & implementation).
However, using the result of a comma expression is already an error.
This DIP does not depend on tuple literals, so there is no reason to delay this DIP for them.

Currently only an [unpacking declaration](https://dlang.org/spec/declaration.html#unpacking-declarations)
can destructure a tuple. A tuple pattern component must
be either a new variable declaration or a nested pattern. Sometimes a new variable for one
or more components is not actually needed, as:

- Existing variables/lvalue expressions need assigning to instead. An existing lvalue
  may be accessible from scopes outside the current function (e.g. on the heap), and
  it is often impractical to change such scopes to use a (new) local variable.
- The programmer may wish to avoid polluting the scope and reuse
  an existing variable when appropriate.

Without unpacking assignments, either:

- Unnecessary extra variables are used, which then have to be assigned to the actual
  intended lvalues.
- An unpacking declaration is avoided and manual assignments are written instead, which
  is more verbose (particularly when the tuple expression needs a temporary variable).

## Prior Work
These languages support unpacking assignments from a tuple:

### C#
> "A deconstruction can mix existing variables, newly declared variables, and discards in one assignment:"

```c#
string city;
(city, var population) = ("New York City", 8_584_629);
```
- <https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/patterns/deconstruct#deconstruct-tuples>

### Rust
```rs
let (a, b);
(a, b) = (1, 2);
```
- <https://blog.rust-lang.org/2022/02/24/Rust-1.59.0/#destructuring-assignments>
- Implemented RFC: <https://rust-lang.github.io/rfcs/2909-destructuring-assignment.html>

Rust does not allow inline declarations in a tuple pattern assignment.
There may be less need for that, because Rust allows type inference even when a variable
declaration has no initializer.

### TypeScript
```ts
let x: number, y: string;
[x, y] = [5, 'hi'];
```
- <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring>

TypeScript does not allow inline declarations in a tuple pattern assignment.

## Description
An [*UnpackDeclaration*](https://dlang.org/spec/declaration.html#unpacking-declarations)
has a tuple pattern which is a list of pattern components surrounded by parentheses.
A pattern component is a declaration, an identifier (when an outside storage class applies),
or a nested pattern.

This DIP adds an *UnpackStatement*, which is equivalent to an *UnpackDeclaration* without
leading *StorageClasses*, except a pattern component may also be an *Expression*.
Each expression component must be an lvalue.

```d
import std.typecons : tuple;

auto t = tuple(1, "hi");

void f() {
    (int i, string s) = t; // unpack declaration

    // equivalent to:
    int i;
    string s;
    (i, s) = t; // unpack statement
}
```
The above unpack statement is equivalent to:
```d
    auto ref __tmp = t;
    i = __tmp[0];
    s = __tmp[1];
```
The `__tmp` variable declaration may be avoided when using `t` directly is equivalent.

An *UnpackStatement* can mix declaration and expression pattern components:
```d
void g() {
    string[1] b;
    auto t = tuple(1, "2");

    (ref a, b[0]) = t; // declare `a`, assign to `b[0]`
    a++;
    assert(t[0] == 2);
    assert(b[0] == "2");
}
```
A pattern component can be nested:
```d
void h() {
    string d;
    double e;
    (int c, (d, e)) = tuple(1, tuple("4", 3.0));
    assert(c == 1);
    assert(d == "4");
    assert(e == 3.0);

    int i;
    ref get() => i;
    (auto (j, k), get()) = tuple(tuple("5", 3.0), 1);
    assert(j == "5");
    assert(k == 3.0);
    assert(i == 1);
}
```

Just like an *UnpackDeclaration*:

* Only a type which is (or implicitly converts to) a value sequence can be unpacked.
* Each pattern component must be compatible with the type of the corresponding element
  being unpacked.
* No pattern component can specify `static` or `enum` storage classes.

### Statement vs Expression
The following is not a proposal of this DIP:
```diff
 AssignExpression:
    [...]
+   TupleExpressionPattern = AssignExpression
```

If a *TupleExpressionPattern* was supported, it should be restricted to pattern components
which are either expressions or nested tuple patterns. Nested patterns would have the same
restriction in turn. No variable components would be allowed.

This would be for consistency because D does not allow a variable to be declared in
an expression, especially not a subexpression. For example, this would be inconsistent:

```d
    (auto v1,) = vals, v1 = func(v1);
```

Given this restriction, in order to combine a declaration component and an
expression component in the same pattern, an *UnpackStatement*
would still be needed. Given the latter, it is not clear that supporting *TupleExpressionPattern*
is worthwhile.

If a *TupleExpressionPattern* was added later, *UnpackStatement* has compatible syntax so no
deprecation would be necessary.

### Grammar Changes
An *UnpackStatement* assigns to a *TupleStatementPattern*.
The *TupleStatementPattern* grammar mirrors [*TuplePattern*](https://dlang.org/spec/declaration.html#TuplePattern),
except a pattern may also be an *Expression*:

```diff
 NonEmptyStatementNoCaseNoDefault:
    [...]
+   UnpackStatement
```
```diff
+UnpackStatement:
+   TupleStatementPattern = Expression ;
+
+TupleStatementPattern:
+   ( TupleStatementPatternComponents )
+
+TupleStatementPatternComponents:
+   TupleStatementPatternComponent ,
+   TupleStatementPatternComponent , TupleStatementPatternComponent
+   TupleStatementPatternComponent , TupleStatementPatternComponents
+
+TupleStatementPatternComponent:
+   TuplePatternComponent
+   Expression
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
Copyright (c) 2026 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## History
The DIP Manager will supplement this section with links to forum discsusionss and a summary of the formal assessment.
