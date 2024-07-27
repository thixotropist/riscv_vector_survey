---
title: vmv.v.?
linkTitle: vmv.v.?
weight: 103
---

{{% pageinfo %}}
Vector move
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#1116-vector-integer-move-instructions)

## Overview

Vector move from vector, scalar, or immediate

## Patterns

Used to copy one vector into another, a scalar register into all elements of a vector, or an immediate value into
all elements of a vector

## Examples

* [inline strlen]({{< relref "../patterns/memset" >}})