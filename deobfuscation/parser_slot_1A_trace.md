# Parser slot `0x1A` trace — pass 7

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. Call site confirmed

The second-stage loader invokes:

```lua
reader[0x1A](meta, 0x0, reader[0x25], reader[0x03], D)
```

The call is at source offset approximately `36912`.

Immediately before it, the wrapper advances the cursor with:

```lua
reader[0x03] = reader[0x03] + start
```

where `start` came from `reader[50]()`.

## 2. Direct assignment search result

A source-wide search for the literal slot forms corresponding to decimal 26 found:

- `A[26] = N.copy` inside `m()` — this is a different table (`A`) and is part of VM/runtime setup.
- `A[26] = nil` inside `k()` — again a different table context.
- no direct `reader[26] = function(...)` assignment.
- no direct `reader[0x1A] = ...` assignment.
- the only direct reader-side use is the call `reader[0B11010](...)` at the parser entry.

Therefore the parser function is installed indirectly or obtained through table aliasing/construction; treating the `A[26]` assignments as the parser function would be incorrect.

## 3. Important table-context distinction

The outer initializer is roughly:

```lua
local f, _, A, N, S, r = {}
S, r, N = D:F(r, S, N, f)
h, S = D:w(f, S, h, N, r)
h, S = D:W(f, r, N, h, S)
S = D:k(N, S, f, r)
S = D:m(S, N, f, r)
S = D:t(S, f)
S = D:yo(h, S, N, r, f)
S = D:go(N, f, S)
S = D:xo(f, N, S)
S = D:Fo(N, f, S)
```

`m()` explicitly sets `A[26] = N.copy`; in the call from `a()`, this corresponds to the VM state table passed as `A`, not automatically to the reader object used later by `Fo()`.

Likewise `k()` clears `A[26]` in its own table context. Numeric slots are therefore context-dependent and must not be globally mapped by number alone.

## 4. Strongest remaining lead: `Fo()` creates the parser wrapper

`Fo()` contains:

```lua
(f)[0x38] = nil

-- ...

(f)[0x38] = function()
    -- reads through reader state
end

(f)[0x37] = function()
    return D:Po(reader)
end

local parserResult = (function()
    local start = reader[50]()
    local meta = reader[0B110](start)
    reader[0B11] = reader[0x03] + start
    -- state checks
    reader[0B11010](meta, 0x0, reader[0x25], reader[0B11], D)
end)
```

The exact source is more heavily obfuscated, but this establishes that `Fo()` is the immediate construction boundary for the parser object used at the call site.

## 5. Reader primitives surrounding the call

Already recovered reader primitives include:

```lua
reader[0x2A] = function()
    local value = reader[0x8](reader[0x25], reader[0x03])
    reader[0x03] = reader[0x03] + 1
    return D:qo(value)
end

reader[0x2C] = function()
    -- uses the 16-bit buffer reader and advances by 2
end

reader[46] = function()
    local value = reader[17](reader[0x25], reader[0x03])
    reader[0x03] = reader[0x03] + 4
    return value
end
```

`reader[0x25]` is the decoded binary buffer and `reader[0x03]` is its cursor.

## 6. New conclusion

The missing `0x1A` implementation is **not** a simple named function hidden elsewhere in the source. It is most likely produced during the reader/parser table construction, with the obfuscator relying on aliases and state-machine initialization to hide the assignment.

The next useful static target is therefore not another literal search for `26`, but the construction path of the object returned by:

```lua
local meta = reader[0B110](start)
```

and the code that establishes the reader's numeric dispatch table before `Fo()` executes.

## 7. Next pass target

Trace these in order:

1. The implementation assigned to reader slot `0B110` (decimal 6), because it produces `meta` immediately before the parser call.
2. Every function called by `Fo()` while constructing/initializing the reader.
3. Any table returned from those functions that is later aliased to `reader`.
4. The first read of the binary record after `reader[0x1A]` is reached.
5. Recover the first tag dispatch as:

```text
tag -> branch -> record length/type -> decoded value -> destination
```

Static analysis only; the recovered payload is not executed.
