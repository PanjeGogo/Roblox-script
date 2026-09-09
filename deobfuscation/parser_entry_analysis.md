# Parser entry / second-stage dispatch — pass 6

Target: `loadervodo.luau.txt` (Luraph v14.8), branch `deobfuscate-vodo`.

## 1. Parser entry recovered

The reader setup creates an anonymous function which first consumes metadata from the current cursor and then invokes a function stored in reader slot `0x1A` (`0B11010`):

```lua
local start = reader[50]()
local meta = reader[0x06](start)

-- ... state checks / cursor update ...

reader[0x1A](meta, 0x0, reader[0x25], reader[0x03], D)
```

The call is at source offset approximately `36912`.

This establishes that the second-stage parser is passed the decoded binary buffer and the current cursor explicitly. It is therefore a binary deserializer/loader boundary, not part of the outer packed-text decoder.

## 2. Important surrounding state transition

Immediately before the parser call, the wrapper computes:

```lua
reader[0x03] = reader[0x03] + start
```

and later continues through a small state machine. This means the fifth argument (`D`) and the cursor are part of parser state; the parser is expected to return or populate data consumed by the VM construction layer.

The wrapper also contains:

```lua
reader[0x2A] = ... -- byte reader
reader[0x2D] = ... -- helper installed later
reader[0x2B] = function() return D:Yo(reader) end
```

The parser is therefore surrounded by the already-established byte/word readers rather than receiving a plain Lua string.

## 3. Reader operations available to the parser

From previous passes:

| Reader | Operation | Cursor advance |
|---|---|---:|
| `0x2A` | `buffer.readu8` | 1 |
| `0x2C` | `buffer.readu16` | 2 |
| `46` | `buffer.readu32` | 4 |
| separate setup | `buffer.readf64` | 8 expected, exact consumer unresolved |

The direct consumers of the decoded buffer use `reader[0x25]` and `reader[0x03]`.

## 4. New structural conclusion

The parser entry has a two-level structure:

```text
LPH& packed text
      |
      v
outer 5-character decoder
      |
      v
98,600-byte binary buffer
      |
      v
reader object
  cursor = 0x03
  buffer = 0x25
      |
      v
metadata/header reader
      |
      v
slot 0x1A parser
      |
      +--> typed records / constants
      +--> prototypes / function metadata
      +--> VM operand data
```

The `05 <length> <ASCII>` records found in the binary buffer are consequently downstream of this parser boundary. Their byte positions must not be treated as constant IDs until the parser's indexing logic is recovered.

## 5. What remains unresolved

The assignment to reader slot `0x1A` is not a direct textual `reader[0x1A] = function(...)` in the source. It is installed through the obfuscated table-construction/state setup. Therefore the exact tag-dispatch body cannot yet be proven from the visible call site alone.

Do **not** equate numeric slot `0x1A` with a particular VM opcode. It belongs to the parser/reader table and is distinct from the central VM dispatch opcode values.

## 6. Next target

Trace the function that populates reader slot `0x1A`, then identify its first byte read and branch conditions. The required output is:

```text
tag -> record format -> bytes consumed -> decoded value -> destination/index
```

After that, correlate destination/index values with the VM operand arrays (`p`, `q`, `b`, `k`, `v`, `J`) and the constant pool.

Static analysis only; no recovered payload is executed.
