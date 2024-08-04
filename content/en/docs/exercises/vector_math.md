---
title: vector math
linkTitle: vector math
weight: 30
---

{{% pageinfo %}}
Inference Engine  applications like `whisper.cpp` make heavy use of tensor math.  These can benefit from
vector instructions, providing perhaps a 3 fold improvement over equivalent scalar code.
{{% /pageinfo %}}

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

