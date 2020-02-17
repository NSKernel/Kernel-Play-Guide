# x86-64: 0x0F Prefix - Two-Byte Opcode

A single byte opcode can offer 256 possibilities of instructions, but we have tens of thousands of instructions and extensions now. There's no magic. The way of achieve such goal is called a Two-Byte Opcode. On x86-64, `0x0F` indicates that the upcomming instruction's opcode consists of two bytes.

## `0x0F` "Prefix"

An x86-64 instruction is encoded like this

| Prefix | Opcode | Other Garbage |
| :--- | :--- | :--- |


You might think that `0x0F` is a prefix. But rather than a _prefix_ defined by x86-64, it's an opcode prefix. So the way it works is that once you read an opcode starts with `0x0F`, you should read the upfollow 2 bytes as the opcode.

A two-byte opcode is formated as

| 0x0F Prefix | Primary Opcode | Secondary Opcode |
| :--- | :--- | :--- |


Take an example of `vmlaunch` from Intel VT, the opcode is `0F 01 C2`, so the encoding is

| 0x0F Prefix | Primary Opcode | Secondary Opcode |
| :--- | :--- | :--- |
| `0F` | `01` | `C2` |

