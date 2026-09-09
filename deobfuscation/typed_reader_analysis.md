# Typed reader / buffer consumer — pass 5

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. Direct consumers of the decoded buffer

Static inspection of the loader shows the decoded buffer is stored in a reader/parser object at slot `0x25` and the current cursor at slot `0x03`.

A byte helper is installed as:

```lua
reader[0x2A] = function()
    local value
    -- obfuscated state-machine wrapper omitted
    value = reader[0x08](reader[0x25], reader[0x03])
    reader[0x03] = reader[0x03] + 1
    return value
end
```

The underlying slot is assigned from `buffer.readu8`:

```lua
reader[0x08] = buffer.readu8
```

Therefore `0x2A` is a one-byte cursor reader.

## 2. 16-bit reader

The loader also defines:

```lua
reader[0x2C] = function()
    local value
    value = reader[0x0A](reader[0x25], reader[0x03])
    reader[0x03] = reader[0x03] + 2
    return value
end
```

The setup path assigns the underlying 16-bit operation from `readu16`. This is independent from the VM-state slot `0x10`, even though that slot is also assigned `_.readu16` elsewhere.

## 3. 32-bit reader

The loader defines a direct helper:

```lua
reader[46] = function()
    local value = reader[17](reader[0x25], reader[0x03])
    reader[0x03] = reader[0x03] + 4
    return value
end
```

The setup path supplies `buffer.readu32` for this operation.

This is the strongest evidence that the binary parser consumes the payload as a cursor-driven binary stream rather than as source text.

## 4. Floating-point reader: unresolved slot alias

Another initialization path contains:

```lua
A[0x17] = N.readf64
```

This proves the parser/VM environment exposes `buffer.readf64`, but the later function:

```lua
ro = function(D, D, f)
    D = f[19](f[0x25], f[0x03])
    return D
end
```

uses slot `19`, not `0x17`. Because this obfuscator reuses numeric indices across separate local tables, it is **not safe yet** to equate `f[19]` with `readf64` solely from the number.

Current status:

- `readu8` consumer: confirmed.
- `readu16` consumer: high confidence/confirmed by call shape and setup.
- `readu32` consumer: confirmed.
- `readf64` availability: confirmed.
- Exact slot used by the floating-point record reader: unresolved.

## 5. Important parser invocation

A nested closure creates a parser value and invokes a function through a table slot with:

```lua
(_[0x1A])(f, 0x0, _[0x25], _[0x03], D)
```

The arguments strongly indicate:

1. parser/output state `f`
2. initial logical index `0`
3. decoded binary buffer
4. current cursor
5. auxiliary parser state `D`

The parser function assigned to that slot is still hidden behind the obfuscated table-construction layer. This is now the primary target for the next pass.

## 6. What can be inferred from the record layout

The binary stream begins with compact numeric bytes (`BE 11 00 56` in the canonical reconstruction), followed by a large cluster of typed records. At least 141 records have the form:

```text
05 <length> <ASCII bytes>
```

Examples include `Path2D`, `GetAsync`, `RequestAsync`, `HttpService`, `Workspace`, `Instance`, `ScreenGui`, and many Luau/Roblox primitive names.

The reader architecture means those records should be interpreted by a dispatch function that first reads a type/tag byte and then consumes a type-specific payload using one of the cursor readers.

## 7. Next exact target

Resolve the function behind the parser invocation and reconstruct its tag dispatch:

```text
tag byte
  -> record type
  -> bytes consumed
  -> decoded value
  -> constant/prototype index
```

Then correlate the resulting indices with VM operands. Do not execute the decoded payload; continue with static analysis only.
