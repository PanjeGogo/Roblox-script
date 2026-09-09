# VM dispatcher reconstruction

Target: `loadervodo.luau.txt` (Luraph Obfuscator v14.8).

This document records the semantic reconstruction of the main `o`-based VM dispatcher. It is a static analysis artifact; the protected payload is not executed.

## Dispatcher model

The central loop increments `V` and uses an instruction value `o` to select an operation. Operands are read from `p[V]`, `q[V]`, `b[V]`, `k[V]`, `v[V]`, and `J[V]`. The VM mutates register/table state in `T`, `u`, `g`, `W`, `L`, `K`, `F`, and closure bookkeeping table `C`.

The dispatcher contains approximately 68 explicit `o == constant` tests in the inspected region, surrounded by range checks. Many range branches are fall-through/default operations, so the numeric opcode is not always visible as a standalone equality test.

## Recovered operations

| Opcode | Decimal | Semantic operation |
|---:|---:|---|
| `0x01` | 1 | `T[p[V]] = http_request` |
| `0x03` | 3 | `W = p[V]` |
| `0x05` | 5 | Conditional branch using `T[b[V]] < v[V]` |
| `0x07` | 7 | `T[p[V]] = T[q[V]] / k[V]` |
| `0x09` | 9 | Set `L=q[V]`, `g=T` |
| `0x0D` | 13 | `T[b[V]] = select` |
| `0x0F` | 15 | `T[p[V]] = T[q[V]] % k[V]` |
| `0x11` | 17 | `W = p[V]` |
| `0x13` | 19 | `T[p[V]] = writefile` |
| `0x17` | 23 | Indexed assignment through `T[q[V]][T[p[V]]] = T[b[V]]` (range branch) |
| `0x19` | 25 | Build/call a captured closure/argument structure from metadata in `v[V]` |
| `0x1B` | 27 | Function call with variable argument/return count |
| `0x1F` | 31 | Default branch in this range loads `D.ia` into `T[b[V]]` |
| `0x21` | 33 | `T[p[V]] = type` (adjacent branch) |
| `0x22` | 34 | String concatenation `g .. W`, then store through `u[L]` |
| `0x23` | 35 | `T[q[V]] = f[0x95]` |
| `0x2B` | 43 | Closure/upvalue cleanup and returns `false, h, h` |
| `0x2D` | 45 | `g -= W`, store to `u[L]` |
| `0x2F` | 47 | `T[p[V]] = S[q[V]][k[V]]` |
| `0x31` | 49 | Load `T` as a call target context; constructs/copies captured argument descriptors |
| `0x33` | 51 | `T[p[V]] = T[b[V]] // v[V]` |
| `0x37` | 55 | `W=b[V]`, `g=g[W]` |
| `0x39` | 57 | `T[q[V]] = T[b[V]] + J[V]` (range branch) |
| `0x3B` | 59 | `L=J[V]`, `g=k[V]` |
| `0x3D` | 61 | `L=b[V]`, `g=Instance` |
| `0x3F` | 63 | `T[q[V]] = warn` |
| `0x41` | 65 | `T[p[V]] = p` |
| `0x42` | 66 | `T[p[V]] = workspace` (adjacent/default branch) |
| `0x47` | 71 | `u=T`, `L=p[V]` |
| `0x4B` | 75 | `g=nil` |
| `0x4F` | 79 | `T[q[V]][J[V]] = k[V]` |
| `0x51` | 81 | `T[p[V]] = Enum` |
| `0x53` | 83 | Resolve a stored closure/environment slot into `T[b[V]]`; adjacent branch loads `game` |
| `0x55` | 85 | Call `T[D](T[D+1], T[D+2])` with stack cleanup |
| `0x59` | 89 | `T[q[V]] = islclosure` |
| `0x5B` | 91 | Call `T[D](T[D+1], T[D+2])` with stack cleanup |
| `0x5D` | 93 | `T[b[V]] = t` |
| `0x5F` | 95 | `g=g[W]`, then set `W=T`, `F=q[V]` |
| `0x61` | 97 | `T[b[V]] = T[q[V]] ~= J[V]` (range branch) |
| `0x63` | 99 | `T[b[V]] = q` |
| `0x65` | 101 | `T[p[V]] = {}` |
| `0x67` | 103 | Call `T[D](T[D+1])` with stack cleanup |
| `0x69` | 105 | Call target at `T[D]` with packed arguments from `f[0x1e]` |
| `0x6B` | 107 | Closure/upvalue cleanup and returns `false, D, D+p[V]-2` |
| `0x6F` | 111 | `T[p[V]] = request` |
| `0x70` | 112 | `T[p[V]] = CFrame` (the `else` branch paired with opcode 111) |
| `0x71` | 113 | `L=p[V]`, `g={}`, `u[L]=g` |
| `0x73` | 115 | `S[q[V]][k[V]] = T[p[V]]` |
| `0x75` | 117 | `W=b[V]` |
| `0x77` | 119 | `L=p[V]`, `g=S`, `W=q[V]` |
| `0x79` | 121 | Length operation: `g=g[W]`, then `g=#g` |
| `0x7B` | 123 | Copy `n[1..q[V]]` into `T` |
| `0x7D` | 125 | `T[p[V]] = T[b[V]] .. v[V]` |
| `0x7F` | 127 | `T[p[V]] = b` |
| `0x82` | 130 | `u = u[L]` |
| `0x84` | 132 | `T[q[V]] = Color3` |
| `0x8C` | 140 | `T[b[V]] = Vector3` |
| `0x92` | 146 | `T[b[V]] = Vector2` |
| `0x9E` | 158 | `T[p[V]] = isfile` |
| `0xA0` | 160 | `T[q[V]] = D.Oa`; adjacent branch loads `readfile` |
| `0xA4` | 164 | `T[p[V]] = T[b[V]] % T[q[V]]` |
| `0xA6` | 166 | `T[q[V]] = next` |
| `0xAA` | 170 | `T[q[V]] = T[b[V]] * T[p[V]]` |
| `0xB6` | 182 | `T[p[V]] = T[b[V]][T[q[V]]]` |
| `0xB8` | 184 | `T[q[V]] = T[p[V]] - T[b[V]]` (paired branch) |
| `0xBA` | 186 | `T[b[V]] = gethui` |
| `0xBC` | 188 | `T[b[V]] = TweenInfo` |
| `0xBE` | 190 | `T[q[V]] = syn` |
| `0xC1` | 193 | `T[q[V]] = UDim2` |
| `0xC3` | 195 | `T[p[V]] = loadstring` |
| `0xC5` | 197 | `g=g[W]`, `u[L]=g` |
| `0xC9` | 201 | `W=nil`; paired/default branch performs register equality comparison |
| `0xD1` | 209 | Call `T[K]()` after truncating higher registers |
| `0xE9` | 233 | `T[b[V]] = rawget` |
| `0xED` | 237 | `T[p[V]] = buffer` |
| `0xEB` | 235 | `u=T`, `L=q[V]`, `g=k[V]` |
| `0xEF` | 239 | Call/cleanup branch involving `K=p[V]` |
| `0xF7` | 247 | `T[q[V]] = D.ua` |
| `0xF9` | 249 | `T[p[V]] = identifyexecutor` |
| `0xFB` | 251 | `T[q[V]] = UDim` |
| `0xFD` | 253 | Set `W=u`, `F=3`, then `W=W[F]` |
| `0xFF` | 255 | `(T[b[V]])[T[q[V]]] = J[V]` |
| `0x101` | 257 | Copy `n[Y..]` into `T` and truncate higher registers |

## Important VM-facing globals

The dispatcher explicitly exposes Roblox/executor primitives to the VM, including `Color3`, `Vector2`, `Vector3`, `CFrame`, `UDim`, `UDim2`, `TweenInfo`, `Enum`, `game`, `workspace`, `Instance`, `request`, `http_request`, `readfile`, `writefile`, `gethui`, `identifyexecutor`, `isfile`, `islclosure`, `iscclosure`, `typeof`, `rawget`, `rawset`, `buffer`, `task`, `warn`, and `loadstring`.

The combination of `request`/`http_request`, filesystem APIs, executor identification, and `loadstring` indicates that the VM is designed to interact with an executor environment and can dynamically load another chunk. This is evidence about capability, not proof of malicious behavior.

## High-value conclusion

The dispatcher is not ordinary application logic; it is a bytecode interpreter. Therefore translating each branch one-for-one into readable source is insufficient to recover the original script. The next deobfuscation stage should recover the instruction stream/constant tables and then lift VM instructions into pseudocode.

`loadstring` (opcode 195) is especially important: when the VM reaches that instruction, it can obtain a Lua/Luau function from reconstructed data. Any reconstructed payload should be inspected statically rather than executed.
