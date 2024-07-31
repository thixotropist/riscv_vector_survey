---
title: structure accesses
linkTitle: structure accesses
weight: 10
---

{{% pageinfo %}}
Some memory operations can be very hard to understand.  This is especially true
when copying some elements of a structure into elements of a different class
of structure.
{{% /pageinfo %}}

>Note: a tutorial on segmented, strided, and indexed loads and stores would be useful,
>      as would a tutorial on the various element width, length, and multiplier mechanics.

This function is one of the more confusing sequences of vector operations.

```text
void FUN_006759d0(undefined8 param_1,undefined8 param_2,undefined4 *param_3)

{
  long lVar1;
  ulong uVar2;
  long lVar3;
  undefined auVar4 [256];
  undefined auVar5 [256];
  undefined auVar6 [256];
  undefined auVar7 [256];
  int iStack_78;
  undefined auStack_74 [28];
  undefined4 auStack_58 [4];
  int *piStack_48;
  undefined uStack_37;
  
  FUN_000e4230(auStack_58,0x28);
  auStack_58[0] = 0x1f0000;
  piStack_48 = &iStack_78;
  uStack_37 = 0x20;
  lVar1 = FUN_0066f66a(param_1,param_2,auStack_58);
  if (lVar1 == 0) {
    uVar2 = minu((long)iStack_78,7);                   // max number of elements is 7
    *param_3 = (int)uVar2;
    if ((long)iStack_78 != 0) {
      lVar1 = vsetvli_e16mf2tama(uVar2 & 0xffffffff);  // SEW=16, LMUL=1/2, VL=min(7, ??)
      auVar4 = vle32_v(auStack_74);                    // EEW=32, EMUL=(EEW/SEW)*LMUL = 1
      lVar3 = (uVar2 & 0xffffffff) - lVar1;
      auVar5 = vncvt_xxw(auVar4);                      // narrowing from 32 bit to 16 bit
      auVar6 = vnsrl_wi(auVar4,0x10);                  // shift right 0x10 bits, zero extend
      auVar4 = vnsrl_wi(auVar4,0x18);                  // shift right 0x18 bits, zero extend
      vsetvli_e8mf4tama(0);                            // SEW=8, LMUL=1/4, VL unchanged
      auVar7 = vncvt_xxw(auVar5);                      // narrow 
      vnsrl_wi(auVar5,8);
      vncvt_xxw(auVar6);
      vncvt_xxw(auVar4);
      vsseg4e8_v(auVar7,param_3 + 1);
      if (lVar3 != 0) {
        vsetvli_e16mf2tama(lVar3);
        auVar4 = vle32_v(auStack_58 + lVar1 + -7);
        auVar5 = vncvt_xxw(auVar4);
        auVar6 = vnsrl_wi(auVar4,0x10);
        auVar4 = vnsrl_wi(auVar4,0x18);
        vsetvli_e8mf4tama(0);
        auVar7 = vncvt_xxw(auVar5);
        vnsrl_wi(auVar5,8);
        vncvt_xxw(auVar6);
        vncvt_xxw(auVar4);
        vsseg4e8_v(auVar7,param_3 + 1 + lVar1);         // segmented stores
      }
    }
  }
  return;
}
```

If we peek into Ghidra's view of the unstripped binary, we get a little more information.

```c
void ecore_mcp_get_temperature_info(undefined8 param_1,undefined8 param_2,undefined4 *param_3)

{
  long lVar1;
  ulong uVar2;
  long lVar3;
  undefined auVar4 [256];
  undefined auVar5 [256];
  undefined auVar6 [256];
  undefined auVar7 [256];
  int iStack_78;
  undefined auStack_74 [28];
  undefined4 auStack_58 [4];
  int *piStack_48;
  undefined uStack_37;
  
  gp = 0xf25590;
  bzero(auStack_58,0x28);
  auStack_58[0] = 0x1f0000;
  piStack_48 = &iStack_78;
  uStack_37 = 0x20;
  lVar1 = ecore_mcp_cmd_and_union(param_1,param_2,auStack_58);
  if (lVar1 == 0) {
    uVar2 = minu((long)iStack_78,7);
    *param_3 = (int)uVar2;
    if ((long)iStack_78 != 0) {
      lVar1 = vsetvli_e16mf2tama(uVar2 & 0xffffffff);
      auVar4 = vle32_v(auStack_74);
      lVar3 = (uVar2 & 0xffffffff) - lVar1;
      auVar5 = vncvt_xxw(auVar4);
      auVar6 = vnsrl_wi(auVar4,0x10);
      auVar4 = vnsrl_wi(auVar4,0x18);
      vsetvli_e8mf4tama(0);
      auVar7 = vncvt_xxw(auVar5);
      vnsrl_wi(auVar5,8);
      vncvt_xxw(auVar6);
      vncvt_xxw(auVar4);
      vsseg4e8_v(auVar7,param_3 + 1);
      if (lVar3 != 0) {
        vsetvli_e16mf2tama(lVar3);
        auVar4 = vle32_v(auStack_58 + lVar1 + -7);
        auVar5 = vncvt_xxw(auVar4);
        auVar6 = vnsrl_wi(auVar4,0x10);
        auVar4 = vnsrl_wi(auVar4,0x18);
        vsetvli_e8mf4tama(0);
        auVar7 = vncvt_xxw(auVar5);
        vnsrl_wi(auVar5,8);
        vncvt_xxw(auVar6);
        vncvt_xxw(auVar4);
        vsseg4e8_v(auVar7,param_3 + 1 + lVar1);
      }
    }
  }
  return;
}
```

The original C code:

```c
enum _ecore_status_t
ecore_mcp_get_temperature_info(struct ecore_hwfn *p_hwfn,
                               struct ecore_ptt *p_ptt,
                               struct ecore_temperature_info *p_temp_info)
{
        struct ecore_temperature_sensor *p_temp_sensor;
        struct temperature_status_stc mfw_temp_info;
        struct ecore_mcp_mb_params mb_params;
        u32 val;
        enum _ecore_status_t rc;
        u8 i;

        OSAL_MEM_ZERO(&mb_params, sizeof(mb_params));
        mb_params.cmd = DRV_MSG_CODE_GET_TEMPERATURE;
        mb_params.p_data_dst = &mfw_temp_info;
        mb_params.data_dst_size = sizeof(mfw_temp_info);
        rc = ecore_mcp_cmd_and_union(p_hwfn, p_ptt, &mb_params);
        if (rc != ECORE_SUCCESS)
                return rc;

        OSAL_BUILD_BUG_ON(ECORE_MAX_NUM_OF_SENSORS != MAX_NUM_OF_SENSORS);
        p_temp_info->num_sensors = OSAL_MIN_T(u32, mfw_temp_info.num_of_sensors,
                                              ECORE_MAX_NUM_OF_SENSORS);
        for (i = 0; i < p_temp_info->num_sensors; i++) {
                val = mfw_temp_info.sensor[i];
                p_temp_sensor = &p_temp_info->sensors[i];
                p_temp_sensor->sensor_location = (val & SENSOR_LOCATION_MASK) >>
                                                 SENSOR_LOCATION_OFFSET;
                p_temp_sensor->threshold_high = (val & THRESHOLD_HIGH_MASK) >>
                                                THRESHOLD_HIGH_OFFSET;
                p_temp_sensor->critical = (val & CRITICAL_TEMPERATURE_MASK) >>
                                          CRITICAL_TEMPERATURE_OFFSET;
                p_temp_sensor->current_temp = (val & CURRENT_TEMP_MASK) >>
                                              CURRENT_TEMP_OFFSET;
        }

        return ECORE_SUCCESS;
}
```

Start with the C source code to understand the Ghidra decompiler view.

* A loop of at most seven iterations exists
* four 8 bit fields are updated in an array of structures, slicing bit fields out of a 32 bit `val` with
  shift and mask operations

The general pattern here involves transfers from one array of structures to another array of structures,
where the compiler knows the maximum number of elements in the array - 7 in this case.  The compiler is guaranteed that
it can hold at least four 32 bit values in a single vector register, given a minimum VLEN of 128 bits.  It *may* be able
to hold eight 32 bit values in a single register if VLEN is at least 256 bits.  For each 32 bit value read four 8 bit values must
be written.

The compiler has chosen to process four elements first, then the trailing elements in a similar block *if* `num_sensors > 4` *and* `VLEN < 256`.
Therefore the loop has been completely unrolled.

The C pseudocode presented in the decompiler window does not provide enough information to understand the full logic.
The annotated listing view helps:

```text
add.uw      a5,a5,zero             ; zero extend low order 32 bits of a5 into a5
vsetvli     a4,a5,e16,mf2,ta,ma    ; a4=VL; SEW=16, LMUL=1/2, tail agnostic, mask agnostic
c.addi4spn  a3,sp,0xc              ; a3 = sp + 0xc
vle32.v     v1,(a3)                ; v1 is loaded from (a3), Effective MUL = 2 * 1/2 = 1
c.addi      s0,0x4
c.sub       a5,a4
vncvt.x.x.w v2,v1                  ; v2 = zext(short(v1))
vnsrl.wi    v3,v1,0x10             ; v3 = zext(short(v1 >> 16))
vnsrl.wi    v1,v1,0x18             ; v1 = zext(short(v1 >> 24))
vsetvli     zero,zero,e8,mf4,ta,ma ; VL=VL, SEW=8, LMUL=1/4, tail agnostic, mask agnostic
vncvt.x.x.w v4,v2                  ; v4 = zext(byte(v2))
vnsrl.wi    v5,v2,0x8              ; v5 = zext(byte(v2>>8))
vncvt.x.x.w v6,v3                  ; v6 = zext(byte(v3))
vncvt.x.x.w v7,v1                  ; v7 = zext(byte(v1))
vsseg4e8.v  v4,(s0)                ; so[0..VL-1] = v4; so[VL..2*VL-1] = v5
                                   ; so[2*VL..3*VL-1] = v6; so[3*VL..4*VL-1] = v7
```

On a 128 bit vector processor hart VL will be max(a5, LMUL * VLEN / SEW) or max(a5, 4).

Note that the narrowing opcodes (`vncvt.x.x.w` and `vnsrl.wi`) use 2*SEW as the width of the input operand
to generate 1*SEW outputs.  `vncvt.x.x.w` is encoded as `vnsrl.wi` with a zero shift.

