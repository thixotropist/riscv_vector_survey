---
title: autovec
linkTitle: autovec
weight: 20
---

{{% pageinfo %}}
Some memory operations can be very hard to understand.  This is often true
when gcc optimizes structure initialization and configuration code.
{{% /pageinfo %}}

>Note: a tutorial on segmented, strided, and indexed loads and stores would be useful,
>      as would a tutorial on the various element width, length, and multiplier mechanics.

This function highlights some of the more interesting vector instructions.

Ghidra's decompiler window - with some manual annotation - shows us:

```c
long FUN_006577f6(long param_1,undefined8 param_2,uint *param_3,long param_4)
{
  ...
  lVar13 = *(long *)(param_1 + 0x2f8) + 0x344;
  lVar7 = 0x280;                                // operating on 0x280 elements
  puVar6 = param_3;
  do {
    lVar10 = vsetvli_e8m8tama(lVar7);           // SEW=8 bits, LMUL=8
    auVar23 = vle8_v(puVar6);
    lVar7 = lVar7 - lVar10;
    puVar6 = (uint *)((long)puVar6 + lVar10);
    vse8_v(auVar23,lVar13);
    lVar13 = lVar13 + lVar10;
  } while (lVar7 != 0);
}
```

This use of LMUL=8 indicates that 8 vector registers are grouped together.  If VLEN=128 bits, then
16 bytes are loaded into each of 8 vector registers for 128 bytes processed per loop iteration.

The actual C source code is:

```c
OSAL_MEMCPY(&p_hwfn->p_dcbx_info->set, params,
                    sizeof(p_hwfn->p_dcbx_info->set));
```

So this vector sequence is an inline variant of `memcpy`.


Here's an implementation of memset(auStack_e8,0,8)

```c
vsetivli_e8mf2tama(8);
vmv_v_i(in_v1,0);
vse8_v(in_v1,auStack_e8);
```

* It's unclear why the `mf2` term is present
* An 8 byte scalar store is not used, perhaps because the compiler can't verify 64 bit alignment for `auStack_e8`

This sequence is more complex

```c
vsetivli_e8mf4tama(4);                     // SEW=8 bits, LMUL=1/4, VL=4
auVar23 = vle8_v((long)param_3 + 0x22a);   // auVar23[*] = load first 4 bytes of parameter
auVar18 = vle8_v(0x817da8);                // auVar18[*] = load 4 bytes opf constant flags
auVar17 = vle8_v((long)param_3 + 0x22e);   // auVar17[*] = load last 4 bytes of parameter
vmv_v_i(in_v1,0);                          // in_v1[*]=(0,0,0,0)
auVar23 = vmsne_vi(auVar23,0);             // auVar23[*] = (auVar23[*] != 0)
auVar23 = vmerge_vvm(in_v1,auVar18,auVar23); // auVar23[*] = auVar23[*] ? auVar18[*] : 0
auVar18 = vle8_v(0x817dac);                // load 4 bytes
in_v0 = vmsne_vi(auVar17,0);               // in_v0[*] = (auVar23[*] == 0)
vsetvli_e8mf4tamu(0);                      // unmasked elements are now unchanged
vmv_s_x(auVar17,0);                        // auVar17[0] = 0; single element move
auVar23 = vor_vv(auVar23,auVar18,in_v0);   // auVar23[*] |= auVar18[*]
auVar23 = vredor_vs(auVar23,auVar17);      // reduction: auVar23[0] = or( 0 , auVar17[*] )
uVar14 = vmv_x_s(auVar23);                 // uVar14 = auVar23[0]
```

This *appears* to be a vector reduction operation with the `or` operator.

* 0x817da8 is the address of the constant 0x08040201
* 0x817dac is the address of the constant 0x80402010
* the two 4 byte loaded vectors are adjacent in memory

Search through the DPDK source code to identify function generating this sequence.  It appears to be:

```c
u8 pfc_map = 0;
for (i = 0; i < ECORE_MAX_PFC_PRIORITIES; i++)
  if (p_params->pfc.prio[i])
    pfc_map |= (1 << i);
```

The RISCV reduction pattern can be confusing, at it applies the operator in parallel (`vor_vv`) over the vector group,
then transversely over the elements (`vredor`) within that accumulator vector register.


The compiler has apparently processed this source code with these steps:

* `ECORE_MAX_PFC_PRIORITIES` is defined as the constant 8.  Therefore no loops are needed.
* 8 bytes are read, using no more than 1/2 of a single vector register assuming VLEN=128
* the expression `(1 << i)` can be lifted outside of the loop into a constant vector of 8 bytes
* the conditional `if (p_params->pfc.prio[i])` can be translated into a mask generation step
* the expression `pfc_map |= ...` implies a reduction operation.
* in general, vector reduction operations work in two phases - accumulate VLEN-wide vector results
  for all but the last elements, then reduce the final vector of VLEN elements into a scalar in vector
  location 0.
    * this likely explains why the 8 byte vectors are loaded as two 4 byte reads.

This unreleased version of C++ appears to use these pragmas:
* use fractional LMUL values to avoid updates to unused portions of the vector registers
* use `vset*ma` for agnostic handling of unmasked elements at the beginning of a vector stanza
  and `vset*mu` for unchanged handling of unmasked elements in the middle of a vector stanza

A C++ STL representation of this optimized code might be:

```c++
// generated as a constant 8 element byte vector at compile time at location 0x817da8
static const int VECTOR_SIZE = 8;
    vector<byte> flags = {byte(0x01), byte(0x02), byte(0x04), byte(0x08),
         byte(0x10), byte(0x20), byte(0x40), byte(0x80)};

// the prio vector to be used in generating the result.  Known to be 8 bytes long
std::vector<byte> prio = {byte(0), byte(1), byte(0),
         byte(3), byte(0), byte(5), byte(0), byte(7)};

// generated mask implemented by vmsne_vi instructions
//  masks are boolean vectors stored in v0
for (int i = 0; i < VECTOR_SIZE; i++) {
        mask[i] = (prio[i] != byte(0)) ? true : false;
}

// use the mask register to select the active flags
// NOTE: implemented with vmerge for the first 4 elements and a masked vor for the last 4 elements
for (int i = 0; i < VECTOR_SIZE; i++) {
        active_flags[i] = mask[i] ? flags[i] : byte(0);
}

// reduce active_flags[i] using the bitwise or operator
byte pfc_map = std::reduce(active_flags.begin(), active_flags.end(),
        *active_flags.begin(),
        [](byte a, byte b){return a|b;});
```

If we wanted to train an AI to recognize patterns like these we might want to
train on the gcc `riscv/rvv/autovec` test suite.  There are over 1800 test files there,
including something fairly close to our exercise code:

```c
#define COND_REDUCTION(TYPE)                                                   \
  TYPE foo##TYPE (TYPE *restrict a, TYPE *restrict b, int loop_size)           \
  {                                                                            \
    TYPE result = 0;                                                           \
    for (int i = 0; i < loop_size; i++)                                        \
      if (b[i] <= a[i])                                                        \
        result += a[i];                                                        \
    return result;                                                             \
  }

COND_REDUCTION (int8_t)
COND_REDUCTION (int16_t)
COND_REDUCTION (int32_t)
COND_REDUCTION (int64_t)
COND_REDUCTION (uint8_t)
COND_REDUCTION (uint16_t)
COND_REDUCTION (uint32_t)
COND_REDUCTION (uint64_t)
```

You would expect different generated sequences for different values of `loop_size`:

* loop_size unknown at compile time
* loop_size * sizeof(TYPE) less than 1 VLEN
* loop_size * sizeof(TYPE) larger than 1 VLEN but small enough to fit within a register group
* loop_size not a multiple of 4

The same function provides examples of vector `slide` and `gather` instructions

```c
      vsetivli_e32mf2tama(2);                     // SEW=32 bits, VL=2
      auVar23 = vle32_v(param_3 + 0x88);          // auVar23 = load 8 bytes
      vsetivli_e8mf2tama(8);                      // SEW=8 bits, VL=8
      auVar17 = vle8_v(0x932dd0);                 // auVar17 = load (3, 2, 1, 0, 7, 6, 5, 4)
      auVar23 = vrgather_vv(auVar23,auVar17);     // auVar23[*] = auVar23[auVar17[*]]
      uStack_dc = (uint)*(byte *)((long)param_3 + 0x216) << 4 |
                  (uint)*(byte *)((long)param_3 + 0x215) << 8 |
                  (uint)*(byte *)(param_3 + 0x85) << 0xc |
                  (uint)*(byte *)((long)param_3 + 0x213) << 0x10 |
                  (uint)*(byte *)(param_3 + 0x84) << 0x1c |
                  (uint)*(byte *)((long)param_3 + 0x211) << 0x18 |
                  (uint)*(byte *)((long)param_3 + 0x217) |
                  (uint)*(byte *)((long)param_3 + 0x212) << 0x14;
      vse8_v(auVar23,auStack_d0);                  // store 8 bytes into auStack_d0
      uVar14 = rev8((long)(int)param_3[0x87]);
      uVar15 = rev8((long)(int)param_3[0x86]);
      uStack_e0 = (uint)*(byte *)((long)param_3 + 0x233) << 4 | uVar8 & 0xffffff0f;
      uStack_d4 = (undefined4)((ulong)uVar14 >> 0x20);
      uStack_d8 = (undefined4)((ulong)uVar15 >> 0x20);
      if ((long)*(int *)(param_1 + 0x10) << 0x2d < 0) {
        vsetivli_e32mf2tama(2);
        auVar17 = vslidedown_vi(auVar23,1);         // auVar17[i] = auVar23[i+1]
        vmv_x_s(auVar17);                           // scalar result missing
        vmv_x_s(auVar23);
```

The original C source code appears to be
```c
for (i = 0; i < ECORE_MAX_PFC_PRIORITIES; i++) {
        bw_map[i] = p_params->ets_tc_bw_tbl[i];
        tsa_map[i] = p_params->ets_tc_tsa_tbl[i];
        /* Copy the priority value to the corresponding 4 bits in the
          * traffic class table.
          */
        val = (((u32)p_params->ets_pri_tc_tbl[i]) << ((7 - i) * 4));
        p_ets->pri_tc_tbl[0] |= val;
}
for (i = 0; i < 2; i++) {
        p_ets->tc_bw_tbl[i] = OSAL_CPU_TO_BE32(p_ets->tc_bw_tbl[i]);
        p_ets->tc_tsa_tbl[i] = OSAL_CPU_TO_BE32(p_ets->tc_tsa_tbl[i]);
}

DP_VERBOSE(p_hwfn, ECORE_MSG_DCB,
            "flags = 0x%x pri_tc = 0x%x tc_bwl[] = {0x%x, 0x%x} tc_tsa = {0x%x, 0x%x}\n",
            p_ets->flags, p_ets->pri_tc_tbl[0], p_ets->tc_bw_tbl[0],
            p_ets->tc_bw_tbl[1], p_ets->tc_tsa_tbl[0],
            p_ets->tc_tsa_tbl[1]);
```

This example shows:

* a relatively long vector stanza with intermediate scalar operations
* gather and slide operations to translate bytes and big/little endian values
* extraction of byte values for logging
* decompiler loss of values when passed into a varadic logging function via stack slots rather than registers.