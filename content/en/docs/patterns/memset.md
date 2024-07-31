---
title: memset
linkTitle: memset
weight: 20
---

{{% pageinfo %}}
The GCC compiler handles many initialization operations with the `setmem` pseudocode RTL operation.  This includes
both `memset` standard library invocations and sequences of `var = constant;` initializers.  The vectorizer translates
as many of these as it can into equivalent vector initializers.  
{{% /pageinfo %}}

* Often the initializer is an immediate constant zero.
* The compiler takes memory alignment into account when generating loads and stores.  Storing a vector of 8 8 bit bytes
  is not the same as storing a single 64 bit doubleword.

## c and n are known at compile time, and n is small

If the fill byte 'c' is known at compile time and if the size `n` is known at compile time and relatively small,
you may see a pattern like this:

```text
; if n=16 bytes
def $void_pointer = memset(void* $src, int $c, size_t $n)
  vsetivli    zero,0x8,e8,mf2,ta,ma  ; vectors are 8 elements of 8 bits each,
                                     ;   using at most half of the vector register
  vmv.v.i     v1,$c                  ; vector v1 is filled with '0' bytes
  c.addi4spn  $tmp,$src,0x8          ; get a pointer to src + 8 
  vse8.v      v1,($tmp)              ; write 8 bytes of zeros into the first 8 bytes of src
  vse8.v      v1,($src)              ; write 8 bytes of zeros into the second 8 bytes of src
  return      $src as $void_pointer
```

### Notes

* additional instructions may be interleaved with the vector instructions above
* this sequence assumes VLEN is at least 128 bits
* this sequence avoids 16 byte vector writes for an unknown reason

## multiple initializations

Complex initialization sequences can be optimized in surprising ways.

This code fragment sets two adjacent 8 byte elements to (0,0), then two adjacent
byte booleans to (0,1).  The compiler has lots of tricks for generating non-zero initializers.

```c
eal_reset_internal_config(struct internal_config *internal_cfg)
{
...
  internal_cfg->hugefile_prefix = NULL;
  internal_cfg->hugepage_dir = NULL;
  internal_cfg->hugepage_file.unlink_before_mapping = false;
  internal_cfg->hugepage_file.unlink_existing = true;
}
```

```text
        00328498 57 70 81 cd     vsetivli zero,0x2,e64,m1,ta,ma   ; vector of 2 64 bit elements
        003284a0 57 31 00 5e     vmv.v.i  v2,0x0                  ; initialize to 0
        003284a8 57 70 50 0c     vsetvli  zero,zero,e8,mf8,ta,ma  ; resize each element to 8 bits, using 1/8 of the register
        003284b0 d7 a0 08 52     vid.v    v1                      ; v1[0] = 0; v1[1]=1; ...
        003284b4 93 07 05 09     addi     a5,param_1,0x90
        003284b8 27 f1 07 02     vse64.v  v2,(a5)                 ; 
        003284bc 93 07 45 01     addi     a5,param_1,0x14
        003284c0 a7 80 07 02     vse8.v   v1,(a5)                 ; store (0,1) aka (false,true) into (a5)
```

This example shows mixed width operations:

 * The first vsetivli instruction sets VL=2, SEW=64, LMUL=1
 * The second vmv.v.i instruction zeros two 64 bit elements in v2 - 0x0000000000000000, 0x0000000000000000
 * The third vsetivli instruction leaves VL unchanged at 2, SEW=8, and LMUL=1/8
 * the fourth vid.v instruction sets v1 to 0x??????00, 0x??????01
 * the sixth vse64.v instruction writes (0x0000000000000000, 0x0000000000000000) into param_1 + 0x90
     * This instruction uses an element width of 64, so the effective multiplier (EMUL) becomes LMUL * (EEW/SEW) = 1
 * the eighth vse8.v instruction writes (0x00, 0x01) into param_1 + 0xa4

## References

* [vsetivli]({{< relref "../opcodes/vsetvli" >}})
* [vmv.v.i]({{< relref "../opcodes/vmv.v._" >}})
* [vse8.v]({{< relref "../opcodes/vse_" >}})
* [vid.v]({{< relref "../opcodes/vid.v" >}})
