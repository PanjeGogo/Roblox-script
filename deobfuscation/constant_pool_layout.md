# Constant-pool layout — pass 2

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. Packed decoder anchor

The loader builds `S[0x24]` and decodes the embedded `LPH&...` blob. The arithmetic is:

```lua
value = (p - 33)
      + (k - 0x21) * 85
      + (J - 33) * 0x1c39
      + (j - 0x21) * 614125
      + (D - 0x21) * 52200625
```

For the source blob as stored in the file, the packed text length is **123,253 characters**. Decoding the 5-character groups gives **24,650 groups / 98,600 bytes**. The reconstructed bytes (the same static reconstruction used in the previous pass) have SHA-256:

`af65314fd5a7369e3217b4873ab8b1514a242edb488a61eaa94f992eb0ddbe61`

This pass intentionally keeps the exact static reconstruction used by the earlier artifacts; the runtime's `string.gsub(..., "z", ...)` preprocessing is treated separately because its escaped replacement needs to be modeled with Luau's exact string-literal semantics before changing the canonical blob hash.

## 2. Strong typed-string record signature

A repeated binary pattern is:

```text
05 <length byte> <ASCII bytes...>
```

where `<length>` exactly equals the following printable ASCII string length. A scan of the reconstructed 98,600-byte blob finds **141** such candidate records.

The pool is concentrated near the beginning of the decoded data. Examples:

| Offset | Type | Length | String |
|---:|---:|---:|---|
| 186 | `05` | 10 | `NextNumber` |
| 304 | `05` | 6 | `Path2D` |
| 315 | `05` | 4 | `pack` |
| 337 | `05` | 3 | `[C]` |
| 342 | `05` | 12 | `getmetatable` |
| 365 | `05` | 3 | `rep` |
| 377 | `05` | 8 | `IsStudio` |
| 398 | `05` | 4 | `Enum` |
| 404 | `05` | 12 | `nfcnormalize` |
| 418 | `05` | 6 | `rshift` |
| 426 | `05` | 6 | `cancel` |
| 436 | `05` | 6 | `concat` |
| 464 | `05` | 5 | `pcall` |
| 471 | `05` | 7 | `lrotate` |
| 524 | `05` | 10 | `RunService` |
| 578 | `05` | 7 | `readu32` |
| 591 | `05` | 6 | `Viewer` |
| 602 | `05` | 8 | `Position` |
| 632 | `05` | 6 | `rawset` |
| 691 | `05` | 7 | `Destroy` |
| 736 | `05` | 17 | `GetTangentOnCurve` |
| 889 | `05` | 4 | `Size` |
| 948 | `05` | 8 | `GetAsync` |
| 1052 | `05` | 9 | `ScreenGui` |
| 1236 | `05` | 78 | `The debug library is required on Luau platforms. Please open a support ticket.` |
| 1501 | `05` | 9 | `PostAsync` |
| 1576 | `05` | 12 | `WaitForChild` |
| 1801 | `05` | 6 | `string` |
| 1821 | `05` | 9 | `Workspace` |
| 1847 | `05` | 7 | `Connect` |
| 1982 | `05` | 9 | `ClassName` |
| 2106 | `05` | 12 | `RequestAsync` |
| 2196 | `05` | 8 | `Instance` |
| 2614 | `05` | 11 | `HttpService` |
| 2690 | `05` | 11 | `GetChildren` |
| 2884 | `05` | 10 | `GetService` |

## 3. Important negative result: offsets are not direct references

A direct scan for little-endian 32-bit values equal to each string-record offset does **not** find useful references. Therefore the string records should **not** currently be modeled as “byte offset constants referenced by raw u32 offsets”.

The more likely models are:

1. a sequential constant table where the VM refers to constants by an ordinal/index;
2. a compact typed-record table whose reader advances through records and returns an internal constant ID;
3. a record-local value/index stored around the `05 <len> <bytes>` payload rather than the record's absolute offset.

This is why the next pass should recover the reader that consumes the pool instead of guessing indices from offsets.

## 4. Binary neighborhood observations

Around the first records, the bytes contain many `0x56` values between structured records. For example, the `NextNumber` record begins at offset 186 after a run of `0x56` bytes, and other records are separated by similarly non-printable/marker-looking bytes.

This strongly suggests that `05` is a **record type/tag**, not merely an accidental byte occurring before ASCII text. The surrounding bytes are likely part of the record stream or adjacent constant metadata.

## 5. Semantic evidence from recovered strings

The string pool contains a coherent mix of:

- Luau/runtime functions: `type`, `tonumber`, `tostring`, `pcall`, `xpcall`, `assert`, `next`, `rawget`, `rawset`, `getmetatable`, `setmetatable`, `coroutine`-related names;
- Luau debug/introspection fields: `source`, `linedefined`, `lastlinedefined`, `currentline`, `short_src`, `namewhat`, `nparams`, `isvararg`, `what`, `getinfo`, `traceback`;
- Roblox objects/services: `Instance`, `Workspace`, `StarterPlayer`, `RunService`, `HttpService`, `Enum`, `Folder`, `ScreenGui`;
- Roblox properties/methods: `Parent`, `Name`, `ClassName`, `Position`, `Size`, `Destroy`, `Clone`, `IsA`, `GetChildren`, `WaitForChild`, `GetService`, `Connect`, `Disconnect`;
- HTTP methods: `GetAsync`, `PostAsync`, `RequestAsync`;
- geometry/UI types and methods: `Vector2`, `Vector3`, `CFrame`, `UDim`, `UDim2`, `TweenInfo`, and curve-related names.

These names establish that the protected program has constants for Roblox/Luau APIs and executor/runtime-adjacent capabilities. They do **not**, by themselves, prove that every listed API is actually called by the payload.

## 6. Next concrete target

The next static pass should locate the exact function represented by `S[0x1D](...)` in the decoder/VM setup. Its call shape is visible in the loader as:

```lua
local D,j,J,k,p = S[0x1D](f, 1, 5)
```

The goal is to determine whether `S[0x1D]` is effectively `string.unpack`, `string.byte`, or a wrapper around one of them, and then identify the reader that consumes the `05 <len> <string>` records.

Once that reader is mapped, we can assign **constant ordinals** to the 141 strings and correlate those ordinals against VM operands (`p/q/b/k/v/J`). That is preferable to guessing from absolute byte offsets.

## Status

- Packed stream: reconstructed statically.
- String-record signature: confirmed with 141 candidates.
- Direct absolute-offset references: not found.
- Constant ordinal/index relationship: **not yet recovered**.
- Payload execution: intentionally not performed.
