The RISCV platform provides an excellent experimental testbed for new instruction sets and new compiler optimizations.  This project will provide
a crude survey of instruction patterns a Ghidra analyst might run into.

For example, the GCC compiler suite is very good at unrolling simple loops into efficient but hard to understand sequences of vector instructions.

We'll start with two Linux ELF programs compiled for RISCV-64 with a recent GCC developmental snapshot.

1. The DPDK network appliance `dpdk-ip_pipeline`, something that has few natural vectors.
2. The `Whisper.cpp` voice to text inference engine application, something that has a great many natural vectors.

The minimal goal is to help Ghidra users recognize patterns in optimized binaries, and to give Ghidra developers as much time as possible to
work the challenge of incorporating those patterns in Ghidra's decompiler and/or emulator views.

The maximal goal is to develop an ML training set of patterns a future Ghidra user might find in debugging AI-assisted network appliance code.

The primary content of this repo is in the form of a Hugo + Docsy static web site, eventually rendered to https://thixotropist.github.io/riscv_vector_survey

This project is a sister project to [Ghidra Import Tests](https://github.com/thixotropist/ghidra_import_tests), which collects Linux binaries that may stress-test Ghidra's
current import capabilities.

Any invocation of Ghidra referenced in this project will likely be using a [Ghidra fork](https://github.com/thixotropist/ghidra).
This form tracks recent RISCV Instruction Set Extensions as committed to the binutils project repository.
