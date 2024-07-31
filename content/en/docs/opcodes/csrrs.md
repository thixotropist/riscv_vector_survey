---
title: csrrs
linkTitle: csrrs
weight: 103
---

{{% pageinfo %}}
Read vector context from a Control and Status Register
{{% /pageinfo %}}

[Vector Type Register vtype](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#vector-type-register-vtype)

The read-only XLEN-wide vector type CSR, vtype provides the default type used to interpret the contents of the vector register file, and can only be updated by vset{i}vl{i} instructions. The vector type determines the organization of elements in each vector register, and how multiple vector registers are grouped. The vtype register also indicates how masked-off elements and elements past the current vector length in a vector result are handled.

[Vector Length Register vl](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#35-vector-length-register-vl)

The vl register holds an unsigned integer specifying the number of elements to be updated with results from a vector instruction, as further detailed in Section Prestart, Active, Inactive, Body, and Tail Element Definitions.

[Vector Byte Length vlenb](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#36-vector-byte-length-vlenb)

The XLEN-bit-wide read-only CSR vlenb holds the value VLEN/8, i.e., the vector register length in bytes.

[Vector Start Index CSR vstart](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#37-vector-start-index-csr-vstart)


The XLEN-bit-wide read-write vstart CSR specifies the index of the first element to be executed by a vector instruction, as described in Section Prestart, Active, Inactive, Body, and Tail Element Definitions.

Normally, vstart is only written by hardware on a trap on a vector instruction, with the vstart value representing the element on which the trap was taken (either a synchronous exception or an asynchronous interrupt), and at which execution should resume after a resumable trap is handled.

[Vector Fixed-Point Rounding Mode Register vxrm](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#38-vector-fixed-point-rounding-mode-register-vxrm)

## Example

```text
csrrs       $offset,vl,zero            ; determine the number of elements read
```