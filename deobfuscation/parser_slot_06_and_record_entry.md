# Pass 8 — Slot 0x06 and Record Parser Entry

Branch: `deobfuscate-vodo`

## Confirmed call site

Inside `Fo()` the reader/parser setup contains:

```lua
local start = reader[50]()
local meta = reader[0B110](start) -- decimal 6
reader[0B11] = reader[0x03] + start
reader[0B11010](meta, 0x0, reader[0x25], reader[0B11], start)
```

Here `0B110` is decimal **6**, while `0B11010` is decimal **26**.

## Important correction

The table passed as `reader` is the outer `f` created by `F()`, not the numeric state returned by `W()`.

The construction chain in `a()` is approximately:

```text
F(...) -> outer f is initialized
w(f, ...) -> installs low-level binary helpers
W(f, ...) -> installs readu16/readu32 state helpers
 t(..., f) -> installs string/substring helpers
 yo(..., f) -> installs byte/bit helpers and builds decoded buffer
 go(..., f, ...) -> installs parser utility helpers
 xo(f, ...) -> installs additional parser/VM helpers
 Fo(..., f, ...) -> invokes reader[6] and reader[26]
```

This resolves an earlier ambiguity about which table context slot `0x06` belongs to.

## Slot 0x06

A direct source-wide search does **not** find a textual assignment equivalent to:

```lua
reader[6] = ...
reader[0x06] = ...
reader[0B110] = ...
```

The only direct low-index assignment found for `0x06` is inside `ig()`:

```lua
f[0x6] = S
```

but that function operates on a different VM/record-state table and therefore must not be equated with the `reader[6]` used by `Fo()`.

Likewise:

```lua
A[26] = N.copy
A[26] = nil
```

occur in different table contexts and are not evidence that `reader[26]` is assigned there.

## Strong semantic identification of slot 0x06

`yo()` creates the decoded binary buffer using the following code:

```lua
(S)[0x24] = function(D)
    D = S[0B11111](D, "z", "\\33\\z  \\33!!\\x21")
    local f,A = #D - 0X4, 0
    local N = S[6]((f / 0x5) * 4)
    local r = {}
    for h = 5,f,5 do
        local f = S[11](D,h,h+0X4)
        h = r[f]
        if not h then
            local D,j,J,k,p = S[0X1D](f,0B1,0B101)
            local v = (p-33)+(k-0X21)*85+(J-33)*0x1c39+(j-0x21)*614125+(D-0x21)*52200625
            h = v
            r[f] = h
        end
        S[0X19](N,A,h)
        A += 0x4
    end
    return N
end
```

Therefore `S[6]` must be a callable **buffer allocation/creation primitive** at this point. The constant pool separately contains the string `create`, and the VM-facing environment contains `buffer`, making `buffer.create` a strong semantic candidate.

This is an identification by use, not yet a proven source-level assignment.

## What this tells us about the next parser

The decoded blob is created at `reader[0x25]` and the cursor is stored in `reader[0x03]`.

`Fo()` then obtains a `start` value from `reader[50]`, obtains metadata through `reader[6](start)`, advances the cursor by `start`, and finally calls `reader[26]` with:

1. metadata
2. zero base/index
3. decoded binary buffer
4. cursor
5. start/length value

Thus slot `0x1A` is the actual **record/bytecode parser entry**, while slot `0x06` is a metadata/header helper immediately before it.

## Current unresolved point

The source does not expose a direct textual assignment for the reader's slot `0x06`. It is therefore likely populated indirectly during the initialization/aliasing path rather than by a simple `reader[6] = function` statement.

The next useful static target is to trace the return values and aliases produced by `w()`, `W()`, `yo()`, `go()`, and `xo()` until the callable stored at slot `6` can be named with source-level confidence. After that, trace the first read performed by slot `26` and map its tag dispatch.

## Safety boundary

This pass is static analysis only. The embedded payload is not executed and no recovered network/persistence behavior is enabled.
