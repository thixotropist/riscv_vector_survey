---
title: vfirst.m
linkTitle: vfirst.m
weight: 108
---

{{% pageinfo %}}
Find-first-set mask bit
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#153-vfirst-find-first-set-mask-bit)

The `vfirst` instruction finds the lowest-numbered active element of the source mask vector that has the value 1 and
writes that element’s index to a GPR. If no active element has the value 1, -1 is written to the GPR.
