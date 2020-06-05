---
title: Generating Proto File For Bangdream
tags:
  - Bangdream
  - unity
  - il2cpp
  - protobuf
  - automation
categories: Hack
date: 2018-09-15 14:04:21
---

This post is encouraged by [esterTion's post](https://estertion.win/2018/04/bang-dream-proto%E7%9B%B8%E5%85%B3/). Before I only extract the proto file by hand, but it gives me a easy way to automation. For what is *proto file* and *protobuf*, please check [Google's documentation](https://developers.google.com/protocol-buffers/)
<!--more-->
## Background
Bangdream (Full name: BanG Dream! Girls Band Party!, Japanese: バンドリ！ ガールズバンドパーティ！) is a rhythm card game released by Bushiroad and developed by Craftegg with Unity. As of today, Unity is becoming more and more popular among game developers. In Unity 4.x.x, there's nearly no protection to C# source code. With [dnSpy](https://github.com/0xd4d/dnSpy) or [ILSpy](https://github.com/icsharpcode/ILSpy) it's very easy to read the code or do some hack.

From Unity 5, a method called "il2cpp" is applied to convert C# bytecode (IL) to native code. If target is Android, a `libil2cpp.so` can be found under `lib` directory. Thanks to Perfare's amazing tool [Il2CppDumper](https://github.com/Perfare/Il2CppDumper), reading the code is as easy as before.

Il2CppDumper will generate some files, if you don't want to read source code with IDA, then just get the `dump.cs` which includes all classes, methods and variables. For game like Bangdream, you can get rich infomations from this file, like AES key and protobuf definitions. I think Craftegg want to avoid unnecessary files, so they use *protobuf-net* to write protobuf definitions in C# code.

## Analyse
The method name in `dump.cs` are very reasonable. The protobuf definition is always stored in class like "GetResponse" or "PostRequest" prefixed by API path. So "SuiteMasterGetResponse" is the class storing the protobuf definition of game master data (or game database). By reading the [esterTion's post](https://estertion.win/2018/04/bang-dream-proto%E7%9B%B8%E5%85%B3/) and [gist code](https://gist.github.com/esterTion/768c646a83a089af9e5fbb83b77a59fc), it seems not hard to extract proto file from the protobuf definition in il2cpp file. The only problem is how to get the Tag number, which can only be found in binary file.

### Read and rewrite code
The original code of esterTion is written in PHP. Firstly it read the `dump.cs` and extract the basic structure of protobuf then generate the valid "proto2" file. The best parts are the regular expressions extracting the class body and ProtoMemberAttribute (=proto message member), but sadly it can only run in PCRE system (like PHP). In Python some grammar is invalid (subroutine and atomic group), it spent me hours to constructure working regex for Python.

Now I get the proto file but with all Tags as hex address. The original `getTag` method is not working. Is the original code wrong? Probably not. esterTion seems to use the Mach-O binary, which has the different code from Android library. It needs some adjustments.

### Get the right Tag
Let's open the `libil2cpp.so` with [hex code editor](https://hexed.it/) and jump to the address. The hex code for members is like: `04 00 90 E5 01 10 A0 E3 ... 04 00 90 E5 02 10 A0 E3 ....` One member begins with `04 00 90 E5`, and the following number keeps changing. Do a verfication and you can find that this is exactly the Tag number you want. Notice that the number is stored in 12 bits, so do not only extract one byte.

But it's only one type. Another type begins with `10 4C 2D E9`, the Tag number is 12 bytes away. So replace the original `getTag` code with this:

```python
# Only for Python 3
def getTag(address):
  offset = address & 0xFFFFFFFF

  prog.seek(offset)
  inst = prog.read(4)
  inst = int.from_bytes(inst, byteorder='little', signed=False)
  if inst == 0xe5900004: # caution! little-endian
    prog.seek(offset + 4)
    return int.from_bytes(prog.read(2), 'little', signed=False) & 0xfff
  elif inst == 0xe92d4c10: # caution! little-endian
    prog.seek(offset + 12)
    return int.from_bytes(prog.read(2), 'little', signed=False) & 0xfff
  else:
    print(hex(inst), hex(address))
```

> A small problem: the Tag number 300(0x12c) is stored as 3915(0xf4b). Can't figure out the reason and made a hardcoding.

## Proto supports map
Proto files now supports map definition, but it will be compiled to a array structure with objects referring the map key and value. For example, the following protos is equivalent:

```
message test {
  map<uint32, string> entries = 1;
}
```

```
message testEntry {
  required uint32 key = 1;
  required string value = 2;
}
message test {
  repeated testEntry entries = 1;
}
```

The original code only supports the second one. The protobuf for Python supports to generate map structure from the first definition. So the first one is optimal for me. Some changes were made to generate the beautiful map definition.

## Final words
You can find the gist code [here](https://gist.github.com/dnaroma/1bfc901d95f777a340fcb615d6a96bd3). I've tested the generated proto file, it fits the game master data. It's time to say goodbye to manual writting proto file.
