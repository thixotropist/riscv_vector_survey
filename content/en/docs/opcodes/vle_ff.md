---
title: vle_ff
linkTitle: vle_ff
weight: 101
---

{{% pageinfo %}}
Vector load 8, 16, 32, 64 bit elements with fault only first
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#77-unit-stride-fault-only-first-loads)

## Overview

Vector load with unit stride (adjacent elements) with page and memory faults suppressed
for reads other than the first element.

## Patterns

Typically found when loading a vector with unknown effective length, and where that vector
might straddle a page boundary causing a page fault.

## Examples

* [inline strlen]({{< relref "../patterns/strlen" >}})