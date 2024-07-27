---
title: strlen
linkTitle: strlen
weight: 10
---

{{% pageinfo %}}
The `strlen` standard library function can be vectorized inline
{{% /pageinfo %}}

Inline invocation of `strlen(char* src)` can look like this:

```text
def $len = strlen($src)
  c.li        $offset, 0
  c.li        $pointer, $src
$1:
  vsetvli     $scratch,zero,e8,m1,ta,ma  ; set element width to 8 bits, tail and mask agnostic
  c.add       $pointer,$offset
  vle8ff.v    $v_tmp,($pointer)          ; attempt to load vector register $v_tmp from memory
  vmseq.vi    $v_tmp,$v_tmp,0x0          ; set mask if element equals immediate value
  csrrs       $offset,vl,zero            ; determine the number of elements actually read
  vfirst.m    $r_tmp,$v_tmp              ; find the first match (the first 0x0) or return -1
  blt         $r_tmp,zero,$1             ; keep searching if no match
  c.add       $pointer,$r_tmp            ; set pointer to the null element
  c.sub       $pointer,$src              ; subtract src start to get length
  return      $pointer as $len
```

The instruction pair `vle8ff.v`..`vfirst.m` within a short loop is probably the best marker for this pattern.
This sequence locates the first null (0x0) byte found after `$src`, returning the offset of that null byte
from `$src`.

## Notes

This `vsetvli` instruction sets the element width to 8 bits, returning the number of such elements that fit into
a vector register in register `s2`.  If the vector size is 128 bits, that would be 16 elements processed in parallel.  However, if the base address is just below a page or segment boundary, the vector load may cause a page
fault or memory access error.  The `vle8ff.v` instruction allows for this by reading fewer bytes and suppressing
exceptions, with the number of elements read retrieved from the `vl` CSR instead of from the register `s2`.

## References

* [vsetvli]({{< relref "../opcodes/vsetvli" >}})
* [vle8ff.v]({{< relref "../opcodes/vle_ff" >}})
* [vmseq.vi]({{< relref "../opcodes/vms__" >}})
* [csrrs $reg,vl,zero]({{< relref "../opcodes/csrrs" >}})
* [vfirst.m]({{< relref "../opcodes/vfirst.m" >}})
