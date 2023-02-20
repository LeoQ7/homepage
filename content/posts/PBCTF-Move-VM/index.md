---
weight: 5
title: "PBCTF 2023 Move VM Writeup"
date: 2022-12-14T17:55:28+08:00
lastmod: 2022-12-14T17:55:28+08:00
draft: false
author: "Qi Qin"
authorLink: "https://leoq7.com"
description: "Writeup for the reverse challenge - Move VM in pbctf 2023"
images: []
# resources:
# - name: "featured-image"
#   src: "featured-image.jpg"


tags: ["Web3", "CTF", "Move"]
categories: ["Web3"]

twemoji: false
lightgallery: true
---

## Preface

This weekend, I participated in the [pbctf 2023](https://ctf.pbctf.com/) and got the first blood for the reverse challenge Move VM. Here is a brief writeup for this challenge.

## Challenge Info

{{< admonition note "Challenge Info" >}}
- Attachment: [message.mv](./message.mv)
- Solves: 6
- Score: 383
{{< /admonition >}}

In this challenge, we are given a `message.mv` file, which is a serialized Move module. As this is a reverse challenge, we need to reverse the module, analyze the algorithm that checks the flag and finally calucate the flag.

## Disassemble the module

It's relatively easy to disassemble Move bytecode by reusing some code from the official [disassembler](https://github.com/aptos-labs/move/tree/main/language/tools/move-disassembler).

```rust
use std::fs;
use move_binary_format::file_format::CompiledModule;
use move_binary_format::binary_views::BinaryIndexedView;
use move_ir_types::location::Spanned;

use move_disassembler::disassembler::{Disassembler, DisassemblerOptions};
use move_bytecode_source_map::mapping::SourceMapping;
fn main() {
    let bytecode_bytes = fs::read("message.mv").expect("Unable to read bytecode file");
    let module = CompiledModule::deserialize(&bytecode_bytes)
            .expect("Module blob can't be deserialized");
    let biview = BinaryIndexedView::Module(&module);
    let mapping = SourceMapping::new_from_view(biview, Spanned::unsafe_no_loc(()).loc).expect("Unable to build dummy source mapping");

    let mut disassembler_options = DisassemblerOptions::new();
    disassembler_options.print_code = true;
    disassembler_options.only_externally_visible = false;
    disassembler_options.print_basic_blocks = true;
    disassembler_options.print_locals = true;

    let disassembler = Disassembler::new(mapping, disassembler_options);
    let dissassemble_string = disassembler.disassemble().expect("Unable to dissassemble");
    println!("{}", dissassemble_string);
}
```

By compiling and running the above code, we can get the move bytecode instructions in a readable format.

## Analyze the algorithm

Before analyzing the algorithm, we need to understand the semantics of the Move bytecode instructions. Since the instructions set is not complicated and the names are self-explanatory, one can easily understand the semantics of each instruction referring to the source code of the [Move interpreter](https://github.com/move-language/move/blob/main/language/move-vm/runtime/src/interpreter.rs).

From the signature of the `check_flag` function, we can see that the function takes a `vector<u8>` as input which should be the flag.

### Format Check

```
B0:
	0: ImmBorrowLoc[0](Arg0: vector<u8>)
	1: StLoc[41](loc40: &vector<u8>)
	2: CopyLoc[41](loc40: &vector<u8>)
	3: VecLen(3)
	4: LdU64(58)
	5: Neq
	6: BrFalse(11)
B1:
	7: LdU8(255)
	8: LdU8(1)
	9: Add
	10: Pop
B2:
    ...
```

The first 2 basic blocks are used to check the length of the input vector. If the length is not 58, the program will jump to the basic block `B2` and trigger an error.

```
B2:
	11: CopyLoc[41](loc40: &vector<u8>)
	12: LdU64(0)
	13: VecImmBorrow(3)
	14: ReadRef
	15: CastU64
	16: LdU8(48)
	17: Shl
	18: CopyLoc[41](loc40: &vector<u8>)
	19: LdU64(1)
	20: VecImmBorrow(3)
	21: ReadRef
	22: CastU64
	23: LdU8(40)
	24: Shl
	25: BitOr
	26: CopyLoc[41](loc40: &vector<u8>)
	27: LdU64(2)
	28: VecImmBorrow(3)
	29: ReadRef
	30: CastU64
	31: LdU8(32)
	32: Shl
	33: BitOr
	34: CopyLoc[41](loc40: &vector<u8>)
	35: LdU64(3)
	36: VecImmBorrow(3)
	37: ReadRef
	38: CastU64
	39: LdU8(24)
	40: Shl
	41: BitOr
	42: CopyLoc[41](loc40: &vector<u8>)
	43: LdU64(4)
	44: VecImmBorrow(3)
	45: ReadRef
	46: CastU64
	47: LdU8(16)
	48: Shl
	49: BitOr
	50: CopyLoc[41](loc40: &vector<u8>)
	51: LdU64(5)
	52: VecImmBorrow(3)
	53: ReadRef
	54: CastU64
	55: LdU8(8)
	56: Shl
	57: BitOr
	58: CopyLoc[41](loc40: &vector<u8>)
	59: CopyLoc[41](loc40: &vector<u8>)
	60: VecLen(3)
	61: LdU64(1)
	62: Sub
	63: VecImmBorrow(3)
	64: ReadRef
	65: CastU64
	66: LdU8(0)
	67: Shl
	68: BitOr
	69: LdU64(29670774015617385)
	70: Xor
	71: LdU64(7049012482871828)
	72: Neq
	73: BrFalse(78)
B3:
	74: LdU8(255)
	75: LdU8(1)
	76: Add
	77: Pop
```

Recall that in the previous code snippet, the flag is stored in `loc40`, so the logic of the code above is checking if $(flag[0] \ll 6 | flag[1] \ll 5 | flag[2] \ll 4 | flag[3] \ll 3 | flag[4] \ll 2 | flag[5] \ll 1 | flag[len(flag)]) \oplus  29670774015617385 == 7049012482871828$, where $flag[i]$ is the $i$-th byte of the flag. If the condition is not satisfied, the program will jump to the basic block `B3` and trigger an error.

By solving the above equation, we can get the flag format: flag[0:6] = 'pbctf{' and flag[-1] = '}'.

### Simple Stack Machine

After the format check, the program will jump to the basic block `B4` and execute the following instructions.

```
B4:
	78: LdConst[0](Vector(U64): [252, 1, 1, 0, 0, 0, 0, 0, 0, 0, 30, 0, 0, 0, 64, 0, 0, 0, ... , 0, 0, 0, 0, 69, 0, 0, 0])
	79: StLoc[5](loc4: vector<u64>)
	80: ImmBorrowLoc[5](loc4: vector<u64>)
	81: StLoc[39](loc38: &vector<u64>)
	82: VecPack(4, 0)
	83: StLoc[6](loc5: vector<u64>)
	84: MutBorrowLoc[6](loc5: vector<u64>)
	85: StLoc[46](loc45: &mut vector<u64>)
	86: LdU64(0)
	87: StLoc[45](loc44: u64)
B5:
	88: CopyLoc[45](loc44: u64)
	89: CopyLoc[39](loc38: &vector<u64>)
	90: VecLen(4)
	91: Lt
	92: BrFalse(539)
B6:
	93: Branch(94)
B7:
	94: CopyLoc[39](loc38: &vector<u64>)
	95: CopyLoc[45](loc44: u64)
	96: VecImmBorrow(4)
	97: ReadRef
	98: StLoc[43](loc42: u64)
	99: CopyLoc[43](loc42: u64)
	100: LdU8(32)
	101: Shr
	102: LdU64(255)
	103: BitAnd
	104: CastU8
	105: StLoc[44](loc43: u8)
	106: MoveLoc[43](loc42: u64)
	107: LdU64(4294967295)
	108: BitAnd
	109: CastU64
	110: StLoc[42](loc41: u64)
	111: CopyLoc[44](loc43: u8)
	112: LdU8(0)
	113: Eq
	114: BrFalse(119)
```

In `B4`, the program loaded a const vector `loc4` and stored its reference in `loc38`. Then the program initialized an empty vector `loc5` and stored its mutable reference in `loc45`. Finally, the program initialized a variable `loc44` to 0.

In `B5`, the program checked if `loc44` is less than the length of `loc38`. If not, the program will jump to `B539` and exit the program. Otherwise, the program will jump to `B6`.

In `B7`, the program loaded the `loc44`-th element of `loc38` as a U64 value and stored the 25-32 bits of the value in `loc43` as a U8 value. Then the program stored the 33-64 bits of the U64 value in `loc41` as a U64 value. Finally, the program checked if `loc43` is 0. If not, the program will jump to `L119`.

After some analysis, we found that the program is a simple stack machine. `loc38` is the program running in the stack machine. `loc44` is the program counter. `loc45` is the stack. `loc43` is the opcode of the current instruction and `loc41` is the argument.

Note that the vector in `loc38` should be interpreted as a vector of U64 values, we should remove the first 2 bytes which is metadata of the vector and then unpack the remaining bytes into U64 values.

```python
struct.unpack("<252Q", bytes(program[2:]))
```

### Reverse the stack machine

It's tedious to reverse the stack machine, so we omit the details here. The following shows its logic in Python.

```python
target = [2209421562, 4020009855, 2511570847, 825727845, 2747945899, 2434240953, 3923412385, 1510700589, 3658116609, 1210550661, 2892531646, 648401340, 2537403886]

def encrypt(x):
    for _ in range(8):
        x = (x>>1)^((((x&1)^0xffffffff)+1)&0xfff63b78)
    return x

flag = ['?']*58

for i in range(6, len(flag), 4):
    assert encrypt(encrypt(encrypt(encrypt(flag[i])^flag[i+1])^flag[i+2])^flag[i+3]) == target[i//4-1]
```

At first glance, the `encrypt` function is a linear function similar to CRC. After some analysis, we found that it actually is a Galois LFSR.

### Solve the LFSR

Take advantage of the property of LFSR, we can solve the LFSR by the code below.

```python
target = [2209421562, 4020009855, 2511570847, 825727845, 2747945899, 2434240953, 3923412385, 1510700589, 3658116609, 1210550661, 2892531646, 648401340, 2537403886]

def inv_lfsr(x):
    for _ in range(8):
        if x & 0x80000000:
            x = ((x ^ 0xfff63b78) << 1) ^ 0x1
        else:
            x <<= 1
    return x

flag = b''

for part in target:
    flag += inv_lfsr(inv_lfsr(inv_lfsr(inv_lfsr(part)))).to_bytes(4, 'little')

print(flag)
'''
b'enjoy haccing blockchains? work for Zellic:pepega:!}'
'''
```

So the flag is `pbctf{enjoy haccing blockchains? work for Zellic:pepega:!}`.