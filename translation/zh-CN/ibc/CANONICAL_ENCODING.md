This document defines a canonical encoding algorithm for data structures which must be encoded in a known fashion for cross-chain verification in the IBC protocol.

The encoding function maps a typed value into a byte array.

### 原始类型

If a value has a primitive type, it is encoded without tags.

#### Numbers

该协议仅处理无符号整数。

`uint32` and `uint64` types are encoded as fixed-size little-endian, with no sign bit.

#### Booleans

Boolean values are encoded as single bits: `0x00` (false) and `0x01` (true).

#### Bytes

Byte arrays are encoded as-is with no length prefix or tag.

### 结构化类型

Structured types with fields are encoded as proto3 `message`s with the appropriate fields.

Canonical `.proto` files are provided with the specification.
