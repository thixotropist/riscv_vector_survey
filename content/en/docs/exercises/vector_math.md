---
title: vector math
linkTitle: vector math
weight: 30
---

{{% pageinfo %}}
Inference Engine  applications like `whisper.cpp` make heavy use of tensor math.  These can benefit from
vector instructions, providing perhaps a 3 fold improvement over equivalent scalar code.
{{% /pageinfo %}}


## 16 bit vector dot product

Profiling `whisper.cpp` with a voice-to-text sample file shows `ggml_vec_dot_f16` to be a hot-spot.  This
function implements a vector dot product using 16 bit floating point values.  It is used heavily in tensor
product calculations.

The ggml scalar source code source code is:

```c
static void ggml_vec_dot_f16(int n, float * restrict s, size_t bs, ggml_fp16_t * restrict x, size_t bx, ggml_fp16_t * restrict y, size_t by, int nrc) 
{
    assert(nrc == 1);
    UNUSED(nrc);
    UNUSED(bx);
    UNUSED(by);
    UNUSED(bs);

    ggml_float sumf = 0.0;
    for (int i = 0; i < n; ++i) {
        sumf += (ggml_float)(GGML_FP16_TO_FP32(x[i])*GGML_FP16_TO_FP32(y[i]));
    }

    *s = sumf;
}
```

Ghidra's decompiler's view of this after some signature alignment and variable renaming is:

```c
void ggml_vec_dot_f16(ulong n,float *s,long x,long y)
{
  bool bVar1;
  ulong uVar2;
  ulong uVar3;
  double dVar4;
  undefined in_v1 [256];
  undefined x_slice [256];
  undefined auVar5 [256];
  undefined y_slice [256];
  ulong in_vlenb;
  
  gp = &__global_pointer$;
  if (0 < (long)n) {
    vsetvli_e64m1tama(0);              // SEW=64 bit
    vmv_v_i(in_v1,0);                  // v1[*] = 0
    uVar2 = in_vlenb >> 3;             // VLENB/8, or the number of elements in a vector register
    auVar5 = vfslide1up_vf(in_v1,0);   // # vd[0]=0, vd[i+1] = in_v1[i]
    do {
      uVar3 = n;
      if (uVar2 < n) {
        uVar3 = uVar2;                  // uVar3 = max(n, uVar2)
      }
      vsetvli_e16mf4tama(uVar3);        // SEW=16 bits
      x_slice = vle16_v(x);
      y_slice = vle16_v(y);
      vsetvli_e64m1tama(0);             // SEW=64 bits
      x = x + (in_vlenb >> 2);
      y = y + (in_vlenb >> 2);
      x_slice = vzext_vf4(x_slice);     // Zero-extend SEW/4 source to SEW destination
      y_slice = vzext_vf4(y_slice);     // Zero-extend SEW/4 source to SEW destination
      x_slice = vsll_vi(x_slice,2);     // vector immediate shift 2
      y_slice = vsll_vi(y_slice,2);     // vector immediate shift 2
      vsetvli_e32mf2tama(uVar3);        // SEW=32 bits
      x_slice = vluxei64_v(&ggml_table_f32_f16,x_slice);  // unordered 64-bit indexed load of SEW data
      y_slice = vluxei64_v(&ggml_table_f32_f16,y_slice);  // unordered 64-bit indexed load of SEW data
      vsetvli_e32mf2tama(0);           // SEW=32 bits (redundant?)
      x_slice = vfmul_vv(x_slice,y_slice); // 32 bit Floating-point multiply
      vmv1r_v(y_slice,auVar5);         // y_slice[i] = auVar5[i] (=0)
      vsetvli_e32mf2tuma(uVar3);       // tail unchanged, AVL=uVar3
      auVar5 = vfwadd_wv(y_slice,x_slice); // Widening FP add, auVar5 is 64 bits wide
      bVar1 = uVar2 < n;
      n = n - uVar2;
    } while (bVar1);
    vsetvli_e64m1tama(0);              // SEW=64 bits
    x_slice = vfmv_sf(0);              // x_slice[0] = 0
    auVar5 = vfredusum_vs(auVar5,x_slice); // Vector Single-Width Floating-Point Sum Reduction 
    dVar4 = (double)vfmv_fs(auVar5);   // dvar4 = auVar5[0]
    *s = (float)dVar4;
    return;
  }
  *s = 0.0;
  return;
}
```

Notes:

* FP16 floats are not yet supported in this RISCV toolchain.  ggml_table_f32_f16 is a 64K table of 32 bit floats
  allowing conversion of an FP16 float into 32 bit floats
* Some of the conversions between 32 and 64 bit floating point are confusing. 

##  quant8 quantized block - with __riscv_v_intrinsic

There are a lot of ways to represent ML weights/parameters.  A simple vector of 16 bit floating point values, in either fp16 or bf16 formats,
is one way.  The `q8_0` quantization is another, which combines a vector of 8 bit integers with single fp16 scale factor.

The ggml C code for this is:

```c
#define QK8_0 32
typedef struct {
    ggml_half d;       // delta - fp16 common scale factor
    int8_t  qs[QK8_0]; // quants
} block_q8_0;
```

The scalar source code for this comes in several flavors, depending on what the compiler believes the target microarchitecture to be:

The scalar version is a simple double loop, with the outer loop iterating over 32 element blocks and the inner loop iterating over
the bytes within each block:

```c
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, size_t bs, const void * restrict vx, size_t bx, const void * restrict vy, size_t by, int nrc) {
    const int qk = QK8_0;
    const int nb = n / qk;

    assert(n % qk == 0);
    const block_q8_0 * restrict x = vx;
    const block_q8_0 * restrict y = vy;

    float sumf = 0.0;

    for (int i = 0; i < nb; i++) {
        int sumi = 0;

        for (int j = 0; j < qk; j++) {
            sumi += x[i].qs[j]*y[i].qs[j];
        }

        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
}
```

If `__riscv_v_intrinsic` is defined (by the compiler, not a header file) the inner loop is absorbed into vector instructions.
This poses a problem, since it only works if the vector registers can hold 32 bytes each.  That's not true for early RVV 1.0 vector
processors.

```c
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, size_t bs, const void * restrict vx, size_t bx, const void * restrict vy, size_t by, int nrc) {
    const int qk = QK8_0;
    const int nb = n / qk;

    assert(n % qk == 0);
    const block_q8_0 * restrict x = vx;
    const block_q8_0 * restrict y = vy;

    float sumf = 0.0;
    size_t vl = __riscv_vsetvl_e8m1(qk);

    for (int i = 0; i < nb; i++) {
        // load elements
        vint8m1_t bx_0 = __riscv_vle8_v_i8m1(x[i].qs, vl);
        vint8m1_t by_0 = __riscv_vle8_v_i8m1(y[i].qs, vl);

        vint16m2_t vw_mul = __riscv_vwmul_vv_i16m2(bx_0, by_0, vl);

        vint32m1_t v_zero = __riscv_vmv_v_x_i32m1(0, vl);
        vint32m1_t v_sum = __riscv_vwredsum_vs_i16m2_i32m1(vw_mul, v_zero, vl);

        int sumi = __riscv_vmv_x_s_i32m1_i32(v_sum);

        sumf += sumi*(GGML_FP16_TO_FP32(x[i].d)*GGML_FP16_TO_FP32(y[i].d));
    }

    *s = sumf;
}
```

This example shows some of the type gymnastics needed if coding using these C vector intrinsic functions.  That type information
would need to be inferred from context if the Ghidra decompiler were to generate compilable C from a vector program.

```text
void ggml_vec_dot_q8_0_q8_0(long n,float *s,block_q8_0 *vx,block_q8_0 *vy)

{
  block_q8_0 *pbVar1;
  block_q8_0 *pbVar2;
  byte *pbVar3;
  undefined8 uVar4;
  byte *pbVar5;
  undefined8 uVar6;
  int iVar7;
  float fVar8;
  undefined auVar9 [256];
  undefined auVar10 [256];
  undefined in_v5 [256];
  
  gp = &__global_pointer$;
  if (0x1f < n) {
    uVar6 = vsetvli_e8m1tama(0x20);
    vsetvli_e32m1tama(uVar6);
    vmv_v_i(in_v5,0);
    fVar8 = 0.0;
    pbVar3 = vx->qs;
    pbVar5 = vy->qs;
    iVar7 = 0;
    do {
      vsetvli_e8m1tama(uVar6);
      auVar10 = vle8_v(pbVar3);
      auVar9 = vle8_v(pbVar5);
      pbVar1 = (block_q8_0 *)(pbVar5 + -2);
      pbVar2 = (block_q8_0 *)(pbVar3 + -2);
      iVar7 = iVar7 + 1;
      auVar9 = vwmul_vv(auVar10,auVar9);       // vector widening multiply
      vsetvli_e16m2tama(0);                    // results now fill two consecutive vector registers
      auVar9 = vwredsum_vs(auVar9,in_v5);      // vector widening reduction
      vsetivli_e32m1tama(0);
      pbVar5 = pbVar5 + 0x22;
      pbVar3 = pbVar3 + 0x22;
      uVar4 = vmv_x_s(auVar9);
      fVar8 = (float)(int)uVar4 *
              (float)(&ggml_table_f32_f16)[(ushort)pbVar1->d] *
              (float)(&ggml_table_f32_f16)[(ushort)pbVar2->d] + fVar8;
    } while (iVar7 < (int)n >> 5);
    *s = fVar8;
    return;
  }
  *s = 0.0;
  return;
}
```

This code will only process half the elements if executed on a processor with 128 bit vector registers.
The scalar source code is actually better, as the compiler can detect that groups of two vector registers are
needed to perform the load operations.


It turns out that this quantization method, q8_0, is deprecated and rarely used. The q5_K quantization is used more
frequently, and its vector code looks like it is designed to run with 128 bit vector registers.

