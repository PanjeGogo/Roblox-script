# Instruction lifting — next static pass

Target: `loadervodo.luau.txt` (Luraph Obfuscator v14.8).

This pass connects the decoded blob format to the VM state/readers and defines the lifting plan. No decoded payload is executed.

## 1. Confirmed decoded container

The visible decoder converts five printable characters into one 32-bit value and writes it with `buffer.writeu32`. The local analysis recovered 24,651 complete groups = 98,604 bytes.

The first bytes are non-textual, so the blob should be treated as packed VM data rather than source text.

## 2. Reader/state model

The surrounding VM installs different buffer readers into state slots. Confirmed examples include:

```lua
state[0x10] = buffer.readu16
state[0x11] = buffer.readu32
state[0x13] = buffer.readu32
```

Another initialization path stores `readf64` in a neighboring state table. This means the same packed buffer can contain fields of different widths/types.

The critical task is therefore not to guess an instruction width globally. Each reader call must be associated with the state field that selects it and with the offset mutation performed by the surrounding helper.

## 3. Correlation with dispatcher

The central dispatcher consumes operand arrays/metadata represented by names such as `p`, `q`, `b`, `k`, `v`, and `J`, while register-like state is held in `T`, `u`, `g`, `W`, `L`, `K`, and `F`.

The already recovered dispatcher gives semantic anchors for the lifting stage. Examples:

- opcode 195 → `loadstring`
- opcode 111 → `request`
- opcode 112 → `CFrame`
- opcode 130 → `u = u[L]`
- opcode 132 → `Color3`
- opcode 140 → `Vector3`
- opcode 146 → `Vector2`
- opcode 158 → `isfile`
- opcode 160 → filesystem read operation
- opcode 170 → multiplication
- opcode 182 → indexed table read
- opcode 233 → `rawget`
- opcode 237 → `buffer`
- opcode 249 → `identifyexecutor`
- opcode 251 → `UDim`
- opcode 255 → indexed assignment

These are semantic anchors, not proof that the encoded instruction byte is numerically identical to the dispatcher comparison value. The VM may transform or remap the instruction value before dispatch.

## 4. Offset-lifting algorithm

The static reconstruction should build a table with these columns:

| Buffer offset | Reader | Raw value | State field | Meaning | Dispatcher opcode |
|---:|---|---|---|---|---|
| TBD | `readu8`/`readu16`/`readu32`/`readf64` | TBD | TBD | TBD | TBD |

For each reader helper:

1. Locate the read call.
2. Identify the buffer argument.
3. Identify the offset argument.
4. Identify the returned value's destination field.
5. Track the offset increment or replacement.
6. Follow the destination field until it becomes `p/q/b/k/v/J` or another dispatcher operand.
7. Record the resulting instruction/constant relationship.

This avoids treating unrelated anti-tamper arithmetic as instruction bytes.

## 5. High-value unresolved fields

The most important unresolved fields from the earlier control-flow analysis are `0x2AC4`, `0x18AF`, and `0x3753`.

`H` explicitly initializes `N[0x2AC4]` under a conditional path. `l` reads `N[0x3753]` and otherwise calls `D:f`. `D:f` performs arithmetic involving `f[0x2AC4] + f[0x18AF]`. The earlier sandbox run reached a `nil + number` failure at this expression.

This should be treated as an analysis clue, not repaired by inserting arbitrary defaults. A real lift must determine which reader/state transition initializes `0x18AF` and whether the sandbox followed an anti-tamper-invalid branch.

## 6. Anti-tamper separation

The VM contains substantial state checks and environment probes. These should be kept separate from recovered application logic:

- executor/environment probes: `identifyexecutor`, `syn`, `gethui`, closure checks, `typeof`
- filesystem probes: `isfile`, `readfile`, `writefile`
- HTTP capability: `request`, `http_request`
- dynamic code capability: `loadstring`
- Roblox objects/types: `game`, `workspace`, `Instance`, `CFrame`, `Vector2`, `Vector3`, `Color3`, `UDim`, `UDim2`, `TweenInfo`, `Enum`

Their presence establishes capability exposed by the VM, but does not by itself establish malicious behavior.

## 7. Current conclusion

The deobfuscation has moved past superficial variable renaming. The next exact recovery target is the mapping from packed-buffer offsets to VM operands and constants. Once enough instruction records are identified, the dispatcher table can be used to lift those records into structured pseudocode.

No claim is made here that the original unobfuscated source has already been recovered exactly. The current artifacts are semantic/static reconstructions of the protected VM.
