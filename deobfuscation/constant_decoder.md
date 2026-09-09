# Constant/blob decoder analysis

Target: `loadervodo.luau.txt` (Luraph Obfuscator v14.8).

This is a static reconstruction of the large encoded data block. No reconstructed payload is executed.

## 1. Encoded block

The VM initialization defines a decoder function and immediately feeds it a long Lua long-string beginning with `LPH&<WE\`.

The relevant logic is equivalent to:

```lua
function decodeBlob(encoded)
    encoded = encoded:gsub("\\33\\z  \\33!!\\x21", "z") -- exact source uses a character-escape form
    local last = #encoded - 4
    local out = buffer.create((last / 5) * 4)
    local offset = 0
    local cache = {}

    for i = 5, last, 5 do
        local chunk = encoded:sub(i, i + 4)
        local value = cache[chunk]
        if not value then
            local a,b,c,d,e = chunk:byte(1, 5)
            value = (e - 33)
                   + (d - 33) * 85
                   + (c - 33) * 85^2
                   + (b - 33) * 85^3
                   + (a - 33) * 85^4
            cache[chunk] = value
        end
        buffer.writeu32(out, offset, value)
        offset += 4
    end

    return out
end
```

The source has obfuscated variable names and numeric spellings, but the arithmetic is a base-85 style five-character-to-32-bit conversion. The byte order is determined by the later `buffer.writeu32` call and therefore should be treated as the VM's native Luau buffer write semantics rather than guessed from the text representation.

## 2. Size recovered statically

The long-string contains **123,257** characters in the local copy. The decoder intentionally ignores the final two characters because its loop processes complete five-character groups starting at character 5.

That produces **24,651 complete groups**, or **98,604 bytes** of decoded buffer data.

The first decoded bytes from applying the visible arithmetic are:

```text
c1 35 54 0a 14 6d 55 37 ee 71 f7 82 d3 2a 6e 6f
80 20 17 a3 8f f5 01 5d f2 59 56 f1 08 e9 54 33
```

These bytes do **not** resemble plain Lua source or a normal uncompressed text header. That is consistent with the decoded buffer being an intermediate VM/packed representation rather than source code.

## 3. Why this is not yet the original source

The visible base-85 layer is only one stage. The surrounding code exposes buffer primitives (`readu8`, `readu16`, `readu32`, `readf64`, etc.) and contains additional stateful routines. In particular, the VM later reads from the decoded buffer through a helper equivalent to:

```lua
function readEncodedValue(state)
    return state.readFunction(state.blob, state.offset)
end
```

The exact reader selected depends on VM state initialized by earlier functions. Therefore simply converting the long string to bytes is insufficient to reconstruct the original Lua source.

## 4. Important related routines

The initialization contains a function equivalent to a byte/word reader setup:

```lua
A[0x17] = N.readf64
A[0x18] = unpack
A[0x19] = nil
A[0x1a] = nil
A[0x1b] = nil
```

and another initialization stage installs `readu32` into the VM state:

```lua
f[0x13] = _.readu32
```

The dispatcher and helper functions subsequently consume data through indexed state fields. This strongly suggests the 98,604-byte buffer is a packed instruction/constant container used by the protected interpreter.

## 5. Decoder characteristics

The visible decoder has a small memoization table keyed by the five-character chunk. This is an optimization, not encryption: repeated encoded words reuse their already-decoded 32-bit value.

The numeric bases are exact powers of 85:

- `85^0 = 1`
- `85^1 = 85`
- `85^2 = 7,225`
- `85^3 = 614,125`
- `85^4 = 52,200,625`

The source uses the equivalent constants `0x1c39`, `614125`, and `52200625`.

## 6. Next lifting target

The next useful step is to map **buffer offsets → decoded VM fields/instructions**. The most promising route is:

1. Identify the exact state fields holding the decoded buffer and current offset.
2. Map every `readu8/readu16/readu32/readf64` call to an offset update.
3. Recover the VM instruction record format.
4. Correlate decoded instruction IDs with the already recovered dispatcher opcode table in `dispatcher.md`.
5. Then lift the instruction stream into readable pseudocode.

This should produce a substantially more meaningful deobfuscation than trying to rename the 169 generated functions one-by-one.

## Safety note

The decoded data is treated as opaque data only. Any later recovered `loadstring` payload should be inspected statically; it should not be executed merely to discover its behavior.
