# Packed buffer / second-stage reader map — pass 4

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. Decoded buffer is confirmed

The embedded `LPH&...` payload is converted by the outer decoder into a `buffer` of **98,600 bytes**.

Canonical reconstruction:

- packed text captured from the decoder: 123,257 characters
- decoder loop starts at Lua position 5 and ends at `#text - 4`
- effective groups: 24,650
- output: 24,650 × 4 = **98,600 bytes**
- SHA-256: `af65314fd5a7369e3217b4873ab8b1514a242edb488a61eaa94f992eb0ddbe61`

The first four bytes of the decoded buffer are `BE 11 00 56`.

## 2. Reader slots established by static dataflow

The loader builds a small buffer-reader object around the decoded payload. The important fields are:

| Slot | Effective operation | Evidence | Confidence |
|---:|---|---|---|
| `0x03` | read cursor | incremented after each buffer read | confirmed |
| `0x08` | `buffer.readu8` | assigned from `f.readu8`; called as `_[0x08](_[0x25], _[0x03])` | confirmed |
| `0x0A` | `buffer.readu16` | helper calls `_[0x0A](_[0x25], _[0x03])`, then advances cursor by `0x02`; `H` installs `f[0x10]=_.readu16` in the VM state, while the parser helper uses its own slot numbering | high confidence; slot numbering is context-dependent |
| `0x11` / `17` | `buffer.readu32` | helper `_[46]` calls `_[17](_[0x25], _[0x03])` and advances cursor by `0x04`; VM setup separately installs `f[0x13]=_.readu32` | confirmed for parser slot `17` |
| `0x17` / `23` | `buffer.readf64` | `k` installs `A[0b10111]=N.readf64` | confirmed |
| `0x2A` | byte-reader helper | repeatedly reads `buffer` at cursor through slot `0x08`, increments cursor, returns byte | confirmed |
| `0x2C` | 16-bit reader helper | reads through slot `0x0A`, increments cursor by 2 | confirmed |
| `46` | 32-bit reader helper | reads through slot `17`, increments cursor by 4 | confirmed |

Important distinction: the obfuscator reuses numeric slot values in different local tables. Do not merge VM-state indices with parser-object indices merely because their numbers differ by one.

## 3. Direct reader helper pseudocode

The source around the parser setup normalizes to:

```lua
reader.readByte = function()
    local value = buffer.readu8(reader.buffer, reader.cursor)
    reader.cursor = reader.cursor + 1
    return value
end

reader.readU16 = function()
    local value = buffer.readu16(reader.buffer, reader.cursor)
    reader.cursor = reader.cursor + 2
    return value
end

reader.readU32 = function()
    local value = buffer.readu32(reader.buffer, reader.cursor)
    reader.cursor = reader.cursor + 4
    return value
end
```

A separate setup path installs `buffer.readf64` for floating-point records.

## 4. The `05 <length> <ASCII>` pattern is a real typed record

The reconstructed buffer contains at least **141** plausible ASCII records with this structure:

```text
05 <one-byte length> <exactly length ASCII bytes>
```

Examples:

```text
05 06 50 61 74 68 32    -> `Path2D`
05 04 70 61 63 6b       -> `pack`
05 03 5B 43 5D          -> `[C]`
05 0C 67 65 74 6D ...   -> `getmetatable`
```

The length byte exactly matches the following string length. This strongly indicates `0x05` is a **string constant/type tag**, rather than an accidental byte sequence.

The first large cluster begins at decoded offset `0x0130` (decimal 304).

## 5. Early string records

The first records found by the structural scan are:

| Offset | Length | Value |
|---:|---:|---|
| 186 | 10 | `NextNumber` |
| 304 | 6 | `Path2D` |
| 315 | 4 | `pack` |
| 337 | 3 | `[C]` |
| 342 | 12 | `getmetatable` |
| 365 | 3 | `rep` |
| 377 | 8 | `IsStudio` |
| 398 | 4 | `Enum` |
| 404 | 12 | `nfcnormalize` |
| 418 | 6 | `rshift` |
| 426 | 6 | `cancel` |
| 436 | 6 | `concat` |
| 464 | 5 | `pcall` |
| 471 | 7 | `lrotate` |
| 524 | 10 | `RunService` |
| 539 | 11 | `linedefined` |
| 561 | 5 | `yield` |
| 578 | 7 | `readu32` |
| 591 | 6 | `Viewer` |
| 602 | 8 | `Position` |
| 632 | 6 | `rawset` |
| 642 | 8 | `isvararg` |
| 678 | 6 | `insert` |
| 691 | 7 | `Destroy` |
| 702 | 4 | `bnot` |
| 708 | 4 | `wait` |
| 717 | 6 | `number` |
| 730 | 4 | `info` |
| 736 | 17 | `GetTangentOnCurve` |
| 768 | 6 | `unpack` |
| 791 | 4 | `copy` |
| 811 | 6 | `xpcall` |
| 872 | 5 | `spawn` |
| 889 | 4 | `Size` |
| 907 | 23 | `The metatable is locked` |
| 932 | 11 | `NextInteger` |
| 948 | 8 | `GetAsync` |
| 969 | 6 | `gmatch` |
| 979 | 7 | `writeu8` |
| 988 | 15 | `lastlinedefined` |
| 1027 | 9 | `FromValue` |
| 1040 | 3 | `sub` |
| 1052 | 9 | `ScreenGui` |
| 1120 | 18 | `DescendantRemoving` |
| 1140 | 7 | `Shuffle` |
| 1155 | 27 | `HumanoidDisplayDistanceType` |
| 1186 | 3 | `abs` |
| 1196 | 12 | `nfdnormalize` |
| 1230 | 4 | `gsub` |
| 1236 | 78 | `The debug library is required on Luau platforms. Please open a support ticket.` |
| 1341 | 5 | `Clone` |
| 1353 | 3 | `IsA` |
| 1358 | 6 | `resume` |
| 1368 | 5 | `Frame` |
| 1391 | 8 | `tonumber` |
| 1433 | 6 | `typeof` |
| 1450 | 7 | `countrz` |
| 1461 | 8 | `EnumType` |
| 1477 | 6 | `lshift` |
| 1487 | 12 | `setmetatable` |
| 1501 | 9 | `PostAsync` |
| 1516 | 9 | `traceback` |
| 1550 | 15 | `AncestryChanged` |
| 1567 | 7 | `nparams` |
| 1576 | 12 | `WaitForChild` |
| 1590 | 4 | `what` |
| 1609 | 11 | `currentline` |
| 1650 | 5 | `Scale` |
| 1660 | 9 | `short_src` |
| 1671 | 6 | `status` |
| 1687 | 7 | `running` |
| 1696 | 4 | `char` |
| 1726 | 8 | `userdata` |
| 1741 | 4 | `=[C]` |
| 1773 | 6 | `select` |
| 1801 | 6 | `string` |
| 1821 | 9 | `Workspace` |
| 1847 | 7 | `Connect` |
| 1858 | 5 | `defer` |
| 1882 | 6 | `assert` |
| 1893 | 9 | `Connected` |
| 1911 | 4 | `type` |
| 1932 | 4 | `byte` |
| 1941 | 8 | `writeu16` |
| 1951 | 10 | `Disconnect` |
| 1973 | 5 | `Enums` |
| 1980 | 9 | `ClassName` |
| 1997 | 6 | `Parent` |
| 2011 | 18 | `GetPositionOnCurve` |
| 2096 | 6 | `rawget` |
| 2104 | 12 | `RequestAsync` |
| 2118 | 4 | `bxor` |
| 2124 | 4 | `wrap` |
| 2164 | 8 | `IsClient` |
| 2179 | 13 | `StarterPlayer` |
| 2194 | 8 | `Instance` |
| 2211 | 6 | `source` |
| 2238 | 4 | `Name` |
| 2253 | 7 | `__index` |
| 2278 | 8 | `FromName` |
| 2314 | 8 | `tostring` |
| 2345 | 5 | `error` |
| 2357 | 5 | `delay` |
| 2370 | 5 | `Value` |
| 2377 | 26 | `GetTangentOnCurveArcLength` |
| 2410 | 11 | `isyieldable` |
| 2426 | 7 | `boolean` |
| 2440 | 9 | `DataModel` |
| 2455 | 27 | `GetPositionOnCurveArcLength` |
| 2495 | 6 | `readu8` |
| 2520 | 4 | `next` |
| 2531 | 6 | `Random` |
| 2612 | 11 | `HttpService` |
| 2635 | 4 | `find` |
| 2652 | 7 | `countlz` |
| 2661 | 6 | `create` |
| 2688 | 11 | `GetChildren` |
| 2710 | 8 | `namewhat` |
| 2724 | 6 | `Folder` |
| 2753 | 5 | `match` |
| 2767 | 14 | `NextUnitVector` |
| 2788 | 6 | `format` |
| 2796 | 9 | `GetLength` |
| 2816 | 7 | `getinfo` |
| 2841 | 4 | `band` |
| 2847 | 16 | `SetControlPoints` |
| 2865 | 8 | `IsServer` |
| 2882 | 10 | `GetService` |
| 2894 | 5 | `close` |
| 2903 | 8 | `EnumItem` |
| 2917 | 8 | `writeu32` |

## 6. What this establishes

The binary stream is not simply a flat string table. The first ~300 bytes contain compact numeric data, then typed values are interleaved with more numeric records. The repeated `0x56`, `0xF8`, `0xEA`, etc. bytes are therefore likely part of the serialized constant/metadata format rather than string delimiters.

The reader helpers prove that the next deobfuscation step should be a **typed-record parser driven by cursor reads**, not another text decoder.

## 7. Next target

Trace the parser functions that call the helpers `0x2A`, `0x2C`, and `46`, and identify the byte/type dispatch around them. For every branch, record:

```text
record offset -> tag -> decoded primitive -> constant ordinal -> VM reference
```

Priority tags:

1. `0x05` string
2. numeric/floating records using `readu32` / `readf64`
3. references/indices using `readu8` / `readu16`
4. table/function/prototype records

No payload execution is required; continue with static analysis only.
