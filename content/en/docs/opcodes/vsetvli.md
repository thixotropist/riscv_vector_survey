---
title: vsetvli, vsetivli
linkTitle: vsetvli
weight: 100
---

{{% pageinfo %}}
Set vector element count and width, plus tail and mask options.
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#sec-vector-config)

## Overview

`vsetvli` and `vsetivli` set context parameters for subsequent vector instructions.  This includes:

* the size of the vector element, e.g. 8, 16, 32, or 64 bits
* the number of vector elements to be processed
* whether to preserve or ignore tail elements in a vector register
* whether to preserve or ignore unmasked elements in a vector register
* whether to bind multiple vector registers into a logical single and larger register

These instructions return the number of elements that can be processed in parallel.

## Patterns

* `vsetvli` takes the number of elements (`vl`) to be processed from an operand register.  It is used
  when that number is unknown at compile time.  If the number operand register in `rs1` is zero, then
  `vl` is set to VLMAX or preserved from the previous `vl`, depending on the destination register.
    * `vsetvli` often begins a vector stanza
    * `vsetvli` often occurs within a vector stanza if type conversion (widening or narrowing) is performed.
* `vsetivli` takes the number of elements (`vl`) to be processed from an immediate value.
  It is used when that number is known at compile time.
    * If `vl` is a small integer, and `vsetivli` is not followed by a loop, then these stanzas are likely
      due to compiler optimization of scalar code doing similar things to two or more adjacent memory locations.

## Examples

### vsetvli

* [inline strlen]({{< relref "../patterns/strlen" >}})