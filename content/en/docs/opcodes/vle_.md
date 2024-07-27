---
title: vle_
linkTitle: vle_
weight: 102
---

{{% pageinfo %}}
Vector load 8, 16, 32, 64 bit elements
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#74-vector-unit-stride-instructions)

## Overview

Vector load with unit stride (adjacent elements).  Vector loads will cause an alignment exception if the equivalent
scalar load would cause one.

Includes:
 * vle8
 * vle16
 * vle32
 * vle64

## Patterns

The simplest of the vector load instructions

## Examples

* [inline strlen]({{< relref "../exercises/structureAccesses" >}})