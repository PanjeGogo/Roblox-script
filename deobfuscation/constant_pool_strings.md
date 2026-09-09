# Constant-pool string recovery

Target: `loadervodo.luau.txt` (Luraph Obfuscator v14.8)

This pass is static only. The decoded packed buffer was reconstructed from the visible five-character → 32-bit decoder; no recovered payload was executed.

## Exact packed-buffer size

The embedded `LPH&...` blob has **123,257 characters**.

The decoder loops with `h = 5, #D-4, 5`, so the exact number of complete 5-character groups is **24,650**, producing **98,600 bytes** through `buffer.writeu32`.

This corrects the earlier working-note estimate of 24,651 groups / 98,604 bytes.

SHA-256 of the reconstructed 98,600-byte buffer:

`af65314fd5a7369e3217b4873ab8b1514a242edb488a61eaa94f992eb0ddbe61`

## Strong constant-record signature

The decoded data contains a repeatable record pattern:

```text
05 <length-byte> <ASCII bytes...>
```

For example:

```text
05 06 Path2D
05 04 pack
05 0c getmetatable
05 03 rep
05 08 IsStudio
05 04 Enum
05 0c nfcnormalize
05 06 rshift
05 06 cancel
05 06 concat
05 05 pcall
05 07 lrotate
05 0a RunService
05 0b linedefined
05 05 yield
05 07 readu32
05 06 Viewer
05 08 Position
```

The length byte matches the ASCII string length exactly. This is strong evidence that these are typed constant records rather than incidental printable bytes.

## Recovered string records

| Buffer offset | Length | Constant |
|---:|---:|---|
| 186 | 10 | `NextNumber` |
| 304 | 6 | `Path2D` |
| 315 | 4 | `pack` |
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
| 1611 | 11 | `currentline` |
| 1650 | 5 | `Scale` |
| 1660 | 9 | `short_src` |
| 1671 | 6 | `status` |
| 1687 | 7 | `running` |
| 1696 | 4 | `char` |
| 1726 | 8 | `userdata` |
| 1741 | 4 | `=[C]` |
| 1801 | 6 | `string` |
| 1821 | 9 | `Workspace` |
| 1847 | 7 | `Connect` |
| 1860 | 6 | `defer` |
| 1884 | 6 | `assert` |
| 1895 | 9 | `Connected` |
| 1943 | 7 | `writeu16` |
| 1953 | 10 | `Disconnect` |
| 1975 | 6 | `Enums` |
| 1982 | 9 | `ClassName` |
| 1999 | 6 | `Parent` |
| 2013 | 18 | `GetPositionOnCurve` |
| 2098 | 6 | `rawget` |
| 2106 | 11 | `RequestAsync` |
| 2126 | 4 | `wrap` |
| 2166 | 8 | `IsClient` |
| 2181 | 13 | `StarterPlayer` |
| 2196 | 8 | `Instance` |
| 2213 | 6 | `source` |
| 2255 | 7 | `__index` |
| 2280 | 8 | `FromName` |
| 2316 | 8 | `tostring` |
| 2347 | 5 | `error` |
| 2359 | 5 | `delay` |
| 2372 | 5 | `Value` |
| 2379 | 26 | `GetTangentOnCurveArcLength` |
| 2412 | 11 | `isyieldable` |
| 2428 | 7 | `boolean` |
| 2442 | 9 | `DataModel` |
| 2457 | 28 | `GetPositionOnCurveArcLength` |
| 2497 | 7 | `readu8` |
| 2522 | 4 | `next` |
| 2533 | 6 | `Random` |
| 2614 | 10 | `HttpService` |
| 2637 | 4 | `find` |
| 2654 | 7 | `countlz` |
| 2663 | 6 | `create` |
| 2690 | 11 | `GetChildren` |
| 2712 | 4 | `namewhat` |
| 2726 | 6 | `Folder` |
| 2755 | 5 | `match` |
| 2769 | 14 | `NextUnitVector` |
| 2790 | 6 | `format` |
| 2798 | 9 | `GetLength` |
| 2818 | 7 | `getinfo` |
| 2849 | 16 | `SetControlPoints` |
| 2867 | 8 | `IsServer` |
| 2884 | 10 | `GetService` |
| 2896 | 5 | `close` |
| 2905 | 8 | `EnumItem` |
| 2919 | 7 | `writeu32` |

## What this changes in the lift

1. The packed buffer is not merely opaque random data: it contains a structured constant area with explicit typed-string records.
2. The first useful constant records begin around offset 186; the concentration of records through roughly offset 3,000 strongly suggests a constant/string pool near the beginning of the packed representation.
3. The constants reveal both Roblox API names and Luau/debug/runtime names. Examples include `RunService`, `Workspace`, `StarterPlayer`, `HttpService`, `ScreenGui`, `Frame`, `Instance`, `GetService`, `WaitForChild`, `RequestAsync`, `GetAsync`, `PostAsync`, and `Path2D`/curve methods.
4. `RequestAsync`, `GetAsync`, and `PostAsync` confirm that HTTP-related API names are present in the constant pool. This is a capability indicator only; it does **not** prove that the final application actually performs network requests.
5. `Viewer`, `Position`, and several Path2D/curve methods suggest a graphical/geometry-related portion of the protected program. This is still an inference from constants, not a recovered source-level call graph.

## Next exact target

The next pass should recover the **constant-record type byte and index/reference relationship** around these records. In particular:

- identify what the `0x05` record tag means;
- locate the table/index that references these strings;
- correlate string indices with VM operand arrays;
- then correlate those operands with dispatcher semantics.

This is a substantially stronger route to source-level lifting than trying to infer a fixed instruction width from the raw bytes.
