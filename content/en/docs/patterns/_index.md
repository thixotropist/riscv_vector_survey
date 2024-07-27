---
title: Patterns
linkTitle: patterns
weight: 20
---

{{% pageinfo %}}
Common vector instruction patterns
{{% /pageinfo %}}

Optimizing compilers often use templates in translating pcode operations and standard functions into sequences of
vector instructions.  Here are some basic patterns.

When those compilers unroll loops with vector instructions you can get more complex patterns.  This is especially true for
structure transforms.
