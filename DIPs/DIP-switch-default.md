# Breakable block construct

| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            | (number/id)                                                     |
| Review Count:   | 0
| Author:         | Nick Treleaven                                                  |
| Implementation: | (links to implementation PR if any)                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")  |

## Abstract

Currently interspersed `if` tests cause each clause to increase indentation:

```d
code1;
if (!test1)
{
	code2;
	if (test2)
	{
		code3;
		...
	}
}
```

This pattern is better expressed with a breakable block:

```d
switch
{
	code1;
	if (test1)
		break;
	code2;
	if (!test2)
		break;
	code3;
	...
}
```

The `switch` construct above has the same semantics as an existing `switch`
statement with only a `default` label:

```d
switch (0)
{
default:
	...
}
```

This DIP avoids the above noisy syntax and encourages use of the
breakable block pattern.

### Links

Optional sections containing links to existing discussions, research papers or any
other supplementary materials.

## Rationale

State a short motivation about the importance and benefits of the proposed
change.  An existing, well-known issue or a use case for an existing projects
can greatly increase the chances of the DIP being understood and carefully
evaluated.

## Description

Detailed technical description of the new semantics.
Language grammar changes (per https://dlang.org/spec/grammar.html)
needed to support the new syntax (or change) must be mentioned.

### Breaking changes / deprecation process

Provide a detailed analysis on how the proposed changes may affect existing
user code and a step-by-step explanation of the deprecation process which is
supposed to handle breakage in a non-intrusive manner. Changes that may break
user code and have no well-defined deprecation process have a minimal chance of
being approved.

### Examples

A more practical explanation of DIP semantics should be given by showing
several examples of its idiomatic application. Inspecting vivid code examples
is usually an easier & quicker way to evaluate the value of a proposal than
trying to reason about its abstract description.

## Copyright & License

Copyright (c) 2017 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Review

Will contain comments / requests from language authors once review is complete,
filled out by the DIP manager - can be both inline and linking to external
document.
