---
title: vse_
linkTitle: vse_
weight: 102
---

{{% pageinfo %}}
Vector store 8, 16, 32, 64 bit elements
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#7-vector-loads-and-stores)

## Overview

Vector store with unit stride (adjacent elements)

## Patterns

Store a vector into main memory.

Notes:
 * vector stores cause alignment faults if an equivalent sequence of scalar stores would fault.

## Examples

* [inline memset]({{< relref "../patterns/memset" >}})