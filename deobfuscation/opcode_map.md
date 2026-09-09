# VM opcode map — observed constants

Target: `loadervodo.luau.txt` (Luraph v14.8).

These opcode numbers are taken directly from the dispatcher branches in the protected source. The exact original opcode names are not recoverable from these constants alone, so the table uses semantic descriptions.

| Opcode | Decimal | Observed operation |
|---:|---:|---|
| `0x82` | 130 | Loads `g = g[W]`, while the adjacent branch handles table/global traversal. |
| `0x84` | 132 | `T[q[V]] = Color3` |
| `0x8C` | 140 | `T[b[V]] = Vector3` |
| `0x92` | 146 | `T[b[V]] = Vector2` |
| `0xC1` | 193 | `T[q[V]] = UDim2` |
| `0xC3` | 195 | `T[p[V]] = loadstring` |
| `0x6F` | 111 | `T[p[V]] = request` |
| `0x70` | 112 | `T[p[V]] = CFrame` |

Other dispatcher operations explicitly observed:

```text
T[q[V]] = tonumber
T[b[V]] = T[p[V]] / T[q[V]]
T[q[V]] = T[p[V]] - T[b[V]]
T[q[V]] = T[b[V]] * T[p[V]]
T[p[V]] = -T[b[V]]
T[p[V]] = T[b[V]] == T[q[V]]
T[q[V]] = T[p[V]] == k[V]
T[p[V]] = request
T[p[V]] = http_request
T[q[V]] = task
T[p[V]] = TweenInfo
T[b[V]] = gethui
T[p[V]] = readfile
T[b[V]] = assert
```

## Bootstrap/global observations

The VM's global environment references Roblox/executor-facing globals including `Vector2`, `Vector3`, `CFrame`, `UDim2`, `Color3`, `TweenInfo`, `gethui`, `readfile`, `request`, and `http_request`. It also references `loadstring`, which is a strong indicator that the VM eventually reconstructs and evaluates another Lua/Luau chunk.

A large encoded string is decoded by helper `S[0x24]`. The decoder uses a five-character base-85-like packing formula and emits four bytes per five-character group. The resulting byte buffer is stored in `S[0x25]`.

The presence of `loadstring` in the dispatcher proves the opcode mapping, but it does not by itself prove what the eventual payload contains. The payload should be treated as untrusted and inspected statically rather than executed.
