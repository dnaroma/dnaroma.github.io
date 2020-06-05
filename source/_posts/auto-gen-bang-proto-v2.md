---
title: Generating Proto File For Bangdream 4.0.0 (V2)
tags:
  - Bangdream
  - unity
  - il2cpp
  - protobuf
  - automation
categories: Hack
date: 2020-03-16 12:06:34
---


You can check [this post](/2018/09/15/auto-gen-bang-proto/) for some background. BangDream now use "on demand" delivery and I can get only arm64 blobs now. They are so different with old armv7 instrcutions so I have to rewrite some code to get correct tag numbers.

At first, use **latest** `il2cppdumper`, or you may have some errors when running the script. I tried immediately with my old script, and all tag numbers are reported *None*. It's pretty annoying but it should have something to do with new instructions of arm64 (aarch64). Now let's check what happend.
<!--more-->
## Disassembly

Like in armv7, protobuf-net codes are also compiled into two kinds of instructions. Use `UserAuthRequest` as an example. `userId` is compiled into:

```
08 04 40 F9 E1 03 00 32 E2 03 1F AA 00 01 40 F9 6D 70 78 14
```

With a [online disassembler](http://shell-storm.org/online/Online-Assembler-and-Disassembler/?opcodes=08+04+40+F9+E1+03+00+32+E2+03+1F+AA+00+01+40+F9+6D+70+78+14&arch=arm64&endianness=little&dis_with_addr=True&dis_with_raw=True&dis_with_ins=True#disassembly) you can get following instructions:

```asm
0x0000000000000000:  08 04 40 F9    ldr x8, [x0, #8]
0x0000000000000004:  E1 03 00 32    orr w1, wzr, #1
0x0000000000000008:  E2 03 1F AA    mov x2, xzr
0x000000000000000c:  00 01 40 F9    ldr x0, [x8]
0x0000000000000010:  6D 70 78 14    b   #0x1e1c1c4
```

and `attestationErrorMsg` is encoded as:

```
F3 0F 1E F8 FD 7B 01 A9 FD 43 00 91 08 04 40 F9 21 01 80 52 ...
```

and processed by [disassembler](http://shell-storm.org/online/Online-Assembler-and-Disassembler/?opcodes=F3+0F+1E+F8+FD+7B+01+A9+FD+43+00+91+08+04+40+F9+21+01+80+52&arch=arm64&endianness=little&dis_with_addr=True&dis_with_raw=True&dis_with_ins=True#disassembly):

```asm
0x0000000000000000:  F3 0F 1E F8    str  x19, [sp, #-0x20]!
0x0000000000000004:  FD 7B 01 A9    stp  x29, x30, [sp, #0x10]
0x0000000000000008:  FD 43 00 91    add  x29, sp, #0x10
0x000000000000000c:  08 04 40 F9    ldr  x8, [x0, #8]
0x0000000000000010:  21 01 80 52    movz w1, #0x9
```

The most important instructions are:

```asm
E1 03 00 32    orr w1, wzr, #1
# or
21 01 80 52    movz w1, #0x9
```

## Instruction Encoding

But how are the immediate encoded? By checking the [reference](https://developer.arm.com/docs/ddi0487/latest/arm-architecture-reference-manual-armv8-for-armv8-a-architecture-profile) and the [instruction encoding](http://kitoslab-eng.blogspot.com/2012/10/armv8-aarch64-instruction-encoding.html) I figured out that `MOVZ` use direct immediate and `ORR` use bitmask immediate. Although they are aarch64 instructions, both of them use 32 bit immediate.

The direct immediate is pretty *direct*, just read the immediate and it's finished:

```
x10x 0010 1xxi iiii iiii iiii iiid dddd
```

But how about bitmask immediate? They are like the rotating encoding of immediate in armv7 but have some different. `ORR` immediate instruction looks like this:

```
x01x 0010 0Nii iiii iiii iinn nnnd dddd
```

`N` together with first *x* (as known as `sf`) refers to bit length (`sf==0 AND N==0` => 32bit or `sf==1` => 64bit), first six binary digits of immediate are `immr` and last six binary digits of immediate are `imms`. You can find the code to decode bitmask immediate in the arm official reference:

```java
// DecodeBitMasks()
// ================
// Decode AArch64 bitfield and logical immediate masks which use a similar encoding structure

(bits(M), bits(M)) DecodeBitMasks(bit immN, bits(6) imms, bits(6) immr, boolean immediate)
  bits(M) tmask, wmask;
  bits(6) levels;

  // Compute log2 of element size
  // 2^len must be in range [2, M]
  len = HighestSetBit(immN:NOT(imms));
  if len < 1 then UNDEFINED;
  assert M >= (1 << len);

  // Determine S, R and S - R parameters
  levels = ZeroExtend(Ones(len), 6);

  // For logical immediates an all-ones value of S is reserved
  // since it would generate a useless all-ones result (many times)
  if immediate && (imms AND levels) == levels then UNDEFINED;

  S = UInt(imms AND levels);
  R = UInt(immr AND levels);
  diff = S - R; // 6-bit subtract with borrow
  esize = 1 << len;
  d = UInt(diff<len-1:0>);

  welem = ZeroExtend(Ones(S + 1), esize);
  telem = ZeroExtend(Ones(d + 1), esize);
  wmask = Replicate(ROR(welem, R));
  tmask = Replicate(telem);

  return (wmask, tmask);
```

But it's pretty hard to understand. `LLVM` gives [code](https://github.com/llvm-mirror/llvm/blob/5c95b810cb3a7dee6d49c030363e5bf0bb41427e/lib/Target/AArch64/MCTargetDesc/AArch64AddressingModes.h#L292) which is way more clear:

```c
static inline uint64_t decodeLogicalImmediate(uint64_t val, unsigned regSize) {
  // Extract the N, imms, and immr fields.
  unsigned N = (val >> 12) & 1;
  unsigned immr = (val >> 6) & 0x3f;
  unsigned imms = val & 0x3f;

  assert((regSize == 64 || N == 0) && "undefined logical immediate encoding");
  int len = 31 - countLeadingZeros((N << 6) | (~imms & 0x3f));
  assert(len >= 0 && "undefined logical immediate encoding");
  unsigned size = (1 << len);
  unsigned R = immr & (size - 1);
  unsigned S = imms & (size - 1);
  assert(S != size - 1 && "undefined logical immediate encoding");
  uint64_t pattern = (1ULL << (S + 1)) - 1;
  for (unsigned i = 0; i < R; ++i)
    pattern = ror(pattern, size);

  // Replicate the pattern to fill the regSize.
  while (size != regSize) {
    pattern |= (pattern << size);
    size *= 2;
  }
  return pattern;
}
```

OK we can now rewrite our code to get correct tag number.

## New `getTag` Function

Attention that in following code we use **Little-Endian**. In our case, `N` and `sf` always equal to 0 so we don't have to care about length, it's fixed to 32 bits.

```python
def getTag(address):
  offset = address & 0xFFFFFFFF

  prog.seek(offset)
  inst = prog.read(4)
  inst = int.from_bytes(inst, byteorder='little', signed=False)
  if inst == 0xf9400408:
    prog.seek(offset + 4)
    inst = int.from_bytes(prog.read(4), 'little', signed=False)
  elif inst == 0xf81e0ff3:
    prog.seek(offset + 16)
    inst = int.from_bytes(prog.read(4), 'little', signed=False)
  else:
    print(hex(inst), hex(address))
    return None
  if inst >> 24 == 0x52:
    return (inst >> 5) & 0xFFFF
  elif inst >> 24 == 0x32:
    retnum = (inst >> 8) & 0xFFFF
    immr = (retnum >> 8) & 0x3F
    imms = (retnum >> 2) & 0x3F
    clz = lambda x: "{:032b}".format(x).index("1")
    _len = 31 - clz((0 << 6) | (~imms & 0x3F))
    _size = 1 << _len
    R = immr & (_size - 1)
    S = imms & (_size - 1)
    ret = (1 << (S+1)) - 1
    for i in range(immr):
      ret = rotr(ret, 32)
    return ret
```

For whole code please see [this gist](https://gist.github.com/dnaroma/1bfc901d95f777a340fcb615d6a96bd3#file-genproto_arm64-py).

Happy hacking!
