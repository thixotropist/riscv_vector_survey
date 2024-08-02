---
title: Glossary
linkTitle: glossary
weight: 80
---

## Vector terms

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#sec-vector-integer)

### Acronyms

AVL
: Application Vector Length, the total number of elements to be processed as a candidate value for vl

EEW
: Each vector operand has an effective element width (EEW) and an effective LMUL (EMUL) that is used to determine the size and location of all the elements within a vector register group. By default, for most operands of most instructions, EEW=SEW and EMUL=LMUL.

ELEN
: The maximum size in bits of a vector element that any operation can produce or consume, ELEN ≥ 8, which must be a power of 2.

EMUL
:  The effective LMUL (EMUL) of each vector operand is determined by the number of registers required to hold the elements. For example, for a widening add operation, such as add 32-bit values to produce 64-bit results, a double-width result requires twice the LMUL of the single-width inputs. 

LMUL
: The vector length multiplier, LMUL, when greater than 1, represents the default number of vector registers that are combined to form a vector register group. Implementations must support LMUL integer values of 1, 2, 4, and 8.

SEW
: Selected element width, 8, 16, 32 or 64 bits.  Each vector register is viewed as being divided into VLEN/SEW elements.

VL
: An unsigned integer specifying the number of elements to be updated with results from a vector instruction

VLEN
: The number of bits in a single vector register, VLEN ≥ ELEN, which must be a power of 2.  Often 128 or 256 bits.

XLEN
: The number of bits in a scalar (x) register, 64 for RISCV-64

### Terms

group
: ...

segment
: The vector load/store segment instructions move multiple contiguous fields in memory to and from consecutively numbered vector registers.

stride
: 