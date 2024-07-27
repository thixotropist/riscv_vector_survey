---
title: vms__.v_
linkTitle: vms__.vi
weight: 102
---

{{% pageinfo %}}
Vector integer compare instruction, generating a vector mask.  Individual opcodes match the pattern
`vms(eq|ne|ltu|lt|leu|le|gtu|gt).(vv|vx|vi)`. These combine an integer comparison operation with the source of the
compared integer(s) - another vector register, a scalar register, or an immediate.
{{% /pageinfo %}}

[Detailed Reference](https://github.com/riscv/riscv-v-spec/blob/master/v-spec.adoc#118-vector-integer-compare-instructions)

## Set if equal

```text
vmseq.vv vd, vs2, vs1, vm  # Vector-vector
vmseq.vx vd, vs2, rs1, vm  # vector-scalar
vmseq.vi vd, vs2, imm, vm  # vector-immediate
```

## Set if not equal

```text
vmsne.vv vd, vs2, vs1, vm  # Vector-vector
vmsne.vx vd, vs2, rs1, vm  # vector-scalar
vmsne.vi vd, vs2, imm, vm  # vector-immediate
```

## Set if less than, unsigned

```text
vmsltu.vv vd, vs2, vs1, vm  # Vector-vector
vmsltu.vx vd, vs2, rs1, vm  # Vector-scalar
```

## Set if less than, signed

```text
vmslt.vv vd, vs2, vs1, vm  # Vector-vector
vmslt.vx vd, vs2, rs1, vm  # vector-scalar
```

## Set if less than or equal, unsigned

```text
vmsleu.vv vd, vs2, vs1, vm   # Vector-vector
vmsleu.vx vd, vs2, rs1, vm   # vector-scalar
vmsleu.vi vd, vs2, imm, vm   # Vector-immediate
```

## Set if less than or equal, signed

```text
vmsle.vv vd, vs2, vs1, vm  # Vector-vector
vmsle.vx vd, vs2, rs1, vm  # vector-scalar
vmsle.vi vd, vs2, imm, vm  # vector-immediate
```

## Set if greater than, unsigned

```text
vmsgtu.vx vd, vs2, rs1, vm   # Vector-scalar
vmsgtu.vi vd, vs2, imm, vm   # Vector-immediate
```

## Set if greater than, signed

```text
vmsgt.vx vd, vs2, rs1, vm    # Vector-scalar
vmsgt.vi vd, vs2, imm, vm    # Vector-immediate
```

>Note: two instructions are not provided directly

``` text
# Set if greater than or equal, unsigned
# vmsgeu.vx vd, vs2, rs1, vm    # Vector-scalar
# Set if greater than or equal, signed
# vmsge.vx vd, vs2, rs1, vm    # Vector-scalar
```

## Examples

* [inline strlen]({{< relref "../patterns/strlen" >}})