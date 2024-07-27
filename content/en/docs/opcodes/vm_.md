---
title: vm_
linkTitle: vm_
weight: 102
---

{{% pageinfo %}}
Vector masked logical instructions.  Individual opcodes match the pattern
`vm(and|namd|andn|or|nor|orn|xnor).mm`. These perform a logical operation on
a vector register using a mask.  Several pseudoinstructions are recognized by
the disassembler.
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#151-vector-mask-register-logical-instructions)


The instructions are:

* vmand.mm vd, vs2, vs1   # vd.mask[i] =   vs2.mask[i] &&  vs1.mask[i]
* vmnand.mm vd, vs2, vs1  # vd.mask[i] = !(vs2.mask[i] &&  vs1.mask[i])
* vmandn.mm vd, vs2, vs1  # vd.mask[i] =   vs2.mask[i] && !vs1.mask[i]
* vmxor.mm  vd, vs2, vs1  # vd.mask[i] =   vs2.mask[i] ^^  vs1.mask[i]
* vmor.mm  vd, vs2, vs1   # vd.mask[i] =   vs2.mask[i] ||  vs1.mask[i]
* vmnor.mm  vd, vs2, vs1  # vd.mask[i] = !(vs2.mask[i] ||  vs1.mask[i])
* vmorn.mm  vd, vs2, vs1  # vd.mask[i] =   vs2.mask[i] || !vs1.mask[i]
* vmxnor.mm vd, vs2, vs1  # vd.mask[i] = !(vs2.mask[i] ^^  vs1.mask[i])

The pseudoinstructions are:

* vmmv.m vd, vs  => vmand.mm vd, vs, vs   # Copy mask register
* vmclr.m vd     => vmxor.mm vd, vd, vd   # Clear mask register
* vmset.m vd     => vmxnor.mm vd, vd, vd  # Set mask register
* vmnot.m vd, vs => vmnand.mm vd, vs, vs  # Invert bits