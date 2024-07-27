---
title: RISCV-64 Vector instruction survey
---

{{< blocks/cover title="RISCV-64 Vector instruction survey" image_anchor="bottom" height="auto" >}}
How to recognize RISCV vector instruction sequences in the Ghidra analyses.
{.mt-5}
{{< /blocks/cover >}}

{{% blocks/section color="primary" %}}

## Introduction

Newer RISCV-64 toolchains and processors have access to many instruction set extensions.  These can
improve program performance, but they can also complicate Ghidra reverse engineering of excecutables.
This survey shows how to recognize RISCV vector instructions in applications that wouldn't be expected
to do a lot of vector math.

{{% /blocks/section %}}