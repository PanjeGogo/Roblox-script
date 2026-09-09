# Next static targets

## Priority 1 — packed-buffer readers

Trace every occurrence of `readu8`, `readu16`, `readu32`, and `readf64` from initialization through their callers. The objective is to recover the buffer-offset transition function and determine the record boundaries.

## Priority 2 — operand construction

Trace writes into the dispatcher operand arrays/tables: `p`, `q`, `b`, `k`, `v`, and `J`. For every write, record its source state and the decoded-buffer offset that supplied it.

## Priority 3 — constant pool

Separate numeric constants, strings, function metadata, and Roblox/executor global references. The dispatcher already gives recognizable anchors such as `loadstring`, `request`, `CFrame`, `Vector3`, `Color3`, and filesystem functions.

## Priority 4 — dynamic-code boundary

Locate the exact point where opcode 195 (`loadstring`) receives its argument. Recover the argument-producing instructions and the resulting string/data statically. Do not execute that resulting chunk.

## Priority 5 — application logic

After enough VM instructions are lifted, group them into ordinary control-flow regions and identify the script's actual features. Anti-tamper/environment checks should remain annotated separately so they are not mistaken for application behavior.

## Status

The dispatcher semantics and blob decoder are documented. The remaining uncertainty is primarily the packed instruction/constant record format and its stateful reader transitions.
