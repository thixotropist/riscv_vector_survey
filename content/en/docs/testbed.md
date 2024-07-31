---
title: RISCV-64 toolchain and deployment environment
linkTitle: testbed
weight: 90
---

We use two RISCV-64 applications to examine the impact of extensions to the RISCV-64 Instruction Set Architecture.
These are:

* `dpdk-ip_pipeline` - a network application framework example where one wouldn't expect a lot of vector math.
* `whisper.cpp` - a Inference Engine voice-to-text application which uses a lot of tensor math

We use the latest RISCV toolchains available to understand what the newer toolchains may soon mean to Ghidra analyses.
At present that means a GCC 15 compiler and `binutils` assembler close to the development tip.  That means most of the
instruction extensions documented here won't show up in products anytime soon.  That's good, because some of these
extensions will demand a lot of preparation time.  That's especially true for RISCV vector instructions, since these
generally do not easily decompile into the typed C language code we expect to see from scalar instructions.

Ghidra normally processes several types of binary inputs, ranging from easiest to hardest

* ELF executables, sharable object libraries, or kernel modules compiled without optimization and not stripped of symbols.
* ELF binaries compiled with full optimization and stripped of symbols not needed by the loader
* Loadable kernels
* Memory snapshots taken from running processes, where dynamic linkages are partially completed.

Our goal is to show how interpretation of RISCV ISA extensions like vector instructions do not materially complicate
the analyst's job.

Notes:

* `dpdk-ip_pipeline` was built and linked against using a GCC-15 RISCV crosscompiler toolchain
  with `-march=rv64gcv_zba_zbb` to ask for vector and bit manipulation ISA extensions
* The executable ran on an Ubuntu 24.04 RISCV VM with all qemu extensions enabled
* Run-time dynamic loading brought the older `libc.so.6` into RAM. It was built with GCC 13.2.0
  with no vector ISA support assumed
* `gcore` provided an as-executed RAM image file
* Ghidra(isa_ext branch) identified 2.7M instructions in the `dpdk-ip_pipeline` + `libc.so` + `libm.so` memory image.
* The `vector_survey.py` analytic identified 77669 vector instructions in the RAM snapshot, or 2.9%  of the total.
