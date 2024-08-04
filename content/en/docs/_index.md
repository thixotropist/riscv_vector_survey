---
title: Documentation
linkTitle: Docs
menu: {main: {weight: 20}}
weight: 20
---

{{% pageinfo %}}
Recognizing vector instruction sequences in large RISCV-64 binaries.
{{% /pageinfo %}}


## Introduction

Newer RISCV-64 toolchains and processors have access to many instruction set extensions.  These can
improve program performance, but they can also complicate Ghidra reverse engineering of executables.
This survey shows how to recognize RISCV vector instructions in several C and C++ linux applications.

### Data Plane Development Kit (DPDK)

Optimizing compilers like GCC can generate vector instruction sequences in code that has little to do with
vectors.  Our first example is `dpdk-ip_pipeline`, a basic framework for network applications.  GCC 14.1
won't find a lot to optimize in network data plane implementations, but it will optimize associated
control plane code. Only a few percent of the total `dpdk-ip_pipeline` instructions are vector instructions.
If Ghidra can not process these correctly up to a third of the total instruction set remains unrecognized.

The survey of `dpdk-ip_pipeline` shows some examples Ghidra users might run into when working on code that
has little to do with mathematics or sequences:

* data initializers - the compiler will aggressively generate vector code for vectors of as few as two
  elements when adjacent memory locations are to be initialized in a similar way.
* stdlib inlined code - calls to standard library functions like `strlen`, `memcpy`, and `memset` can be
  replaced with inline vector instructions.  The inlined code shows a lot of diversity, depending on what
  the compiler can determine of alignment and element numbers.
* loop unrolling - the compiler will often convert a simple loop into a complex loop with fewer iterations.
  If the element count is small enough, and the minimum vector register size is known to be big enough, the
  loop can be converted into a single pass through a vector instruction sequence.  If the loop iterates over
  a sequence of structures the generated vector code can be hard to follow.

### Whisper.cpp

This voice-to-text application is the kind of Inference Engine application that depends heavily on vector
math.  Much of the processing involves vector dot products, often with 16 bit floating point operands.
The optimizing compiler can do a lot with this code, especially if source code breaks the vectors into blocks
that fit entirely within the vector register file.  The compiler will often vectorize the control structures
as well, turning simple run-once scalar sequences into slightly more efficient but far more confusing vector sequences.
