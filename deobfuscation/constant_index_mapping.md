# Constant decoder / index mapping — pass 3

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. `S[0x1D]` is the 5-byte reader used by the packed decoder

The packed-decoder closure contains:

```lua
local D,j,J,k,p = S[0x1D](f, 0b1, 0b101)
```

The five return values are immediately treated as byte values in:

```lua
local v = (p - 33)
        + (k - 0x21) * 85
        + (J - 33) * 0x1c39
        + (j - 0x21) * 614125
        + (D - 0x21) * 52200625
```

This is consistent with `string.byte(f, 1, 5)`: it returns one numeric byte per character and supports the requested start/end positions. The VM helper table independently exposes:

```lua
z = string.byte
```

and another helper installs `A[0x1d] = D.z`, which provides direct corroborating evidence that the obfuscator's helper table aliases this operation through the `string.byte` primitive.

**Conclusion:** `S[0x1D]` should be modeled as **`string.byte`** for this decoder path.

## 2. Other decoder-table entries can be identified from call shape

The same closure contains:

```lua
local f = S[11](D, h, h + 0x4)
```

The result is used as a string key in a memoization table. This matches `string.sub(D, h, h + 4)` and is consistent with the loader exposing:

```lua
La = string.sub
```

Therefore:

| Decoder slot | Effective primitive | Evidence |
|---:|---|---|
| `S[0x1D]` | `string.byte` | Five numeric returns consumed as character bytes; helper alias `z=string.byte` |
| `S[11]` | `string.sub` | `(D,h,h+4)` produces the 5-character group used as a cache key; alias `La=string.sub` |
| `S[6]` | `buffer.create` (high confidence) | Called as `S[6]((f/5)*4)` and result receives 32-bit writes |
| `S[0x19]` | `buffer.writeu32` (high confidence) | Called as `S[0x19](N,A,h)` where `N` is the buffer and `A` advances by 4; the helper table also exposes the obfuscated string `writeu32` |

The last two are marked high-confidence rather than mathematically proven from a single assignment because the table-construction layer is itself obfuscated.

## 3. What the decoder actually produces

The closure therefore has the following readable equivalent:

```lua
function decodePacked(text)
    text = string.gsub(text, "z", "\\33\\z  \\33!!\\x21") -- exact runtime preprocessing still needs Luau-literal modeling

    local last = #text - 4
    local out = buffer.create((last / 5) * 4)
    local cache = {}
    local outOffset = 0

    for pos = 5, last, 5 do
        local group = string.sub(text, pos, pos + 4)
        local value = cache[group]

        if not value then
            local b1, b2, b3, b4, b5 = string.byte(group, 1, 5)
            value = (b5 - 33)
                  + (b4 - 0x21) * 85
                  + (b3 - 33) * 0x1c39
                  + (b2 - 0x21) * 614125
                  + (b1 - 0x21) * 52200625
            cache[group] = value
        end

        buffer.writeu32(out, outOffset, value)
        outOffset += 4
    end

    return out
end
```

The variable ordering above is normalized only for readability. The arithmetic is unchanged.

## 4. Important implication for the constant pool

This decoder does **not** itself parse the `05 <length> <ASCII>` records. It converts the outer five-character packed representation into a binary buffer.

So the 141 typed-string records observed in the reconstructed 98,600-byte stream are a **second-stage format**. The next reader must operate on the returned buffer and interpret its binary record structure.

This resolves the previous uncertainty about the five-character layer: it is a base-85-like byte-packing transform, not the constant ordinal mechanism itself.

## 5. `S[0x24]` / `S[0x25]` relationship

The loader installs the decoder as:

```lua
S[0x24] = function(D) ... return N end
S[0x25] = S[0x24]([=[LPH&...]=])
```

Thus `S[0x25]` is the decoded binary buffer produced from the embedded `LPH&...` text.

The next stage should therefore be traced from **uses of `S[0x25]`** and from the function that consumes the returned buffer, rather than trying to infer constant ordinals directly from the `05` record offsets.

## 6. Current confidence / unresolved point

### Confirmed

- `S[0x1D]` → `string.byte` for the packed decoder.
- `S[11]` → `string.sub` for the packed decoder.
- The five-character groups become 32-bit little-endian values written four bytes at a time.
- `S[0x24]` is the outer packed-text decoder.
- `S[0x25]` receives its decoded binary output.

### High confidence

- `S[6]` → `buffer.create`.
- `S[0x19]` → `buffer.writeu32`.

### Still unresolved

- The exact binary reader that turns the 98,600-byte buffer into typed constants.
- The constant ordinal/index assigned to each of the 141 string records.
- The mapping from those constant ordinals into VM operands/register operations.

## 7. Next target

Trace every read from the decoded buffer (`S[0x25]` / the buffer returned by the decoder), especially calls involving `readu8`, `readu16`, `readu32`, `readf64`, and cursor increments. The goal is to reconstruct the second-stage record reader and emit a table such as:

```text
constant_id -> type -> decoded value -> first VM operand references
```

Do not execute the recovered payload; this pass is static analysis only.
