# Pass 8 — Reader slot `0B110` / decimal 6 trace

Branch: `deobfuscate-vodo`

## Scope

This pass follows the reader-side call immediately before the second-stage parser:

```luau
local start = reader[50]()
local meta = reader[0B110](start) -- decimal 6
reader[0B11] = reader[0X3] + start
reader[0B11010](meta, 0x0, reader[0X25], reader[0B11], D)
```

The goal was to identify the implementation of reader slot `0B110` and connect it to the parser in slot `0B11010` (decimal 26).

## Findings

### 1. `0B110` is decimal 6

The binary literal `0B110` used in `Fo()` is the numeric table key `6`.

The same source contains several other slot-6 expressions, but they are not automatically the same table because Luraph reuses short local names (`f`, `_`, `A`, `N`, etc.) across unrelated functions.

### 2. No direct reader assignment was found

A source-wide static scan found no unambiguous assignment of the form:

```luau
reader[6] = ...
reader[0x6] = ...
reader[0B110] = ...
```

There is one direct `f[0x6] = S` inside `ig()`, but that function is part of the later VM/function-frame construction path and its `f` cannot safely be identified with the reader table created in `Fo()`. Treating it as the reader slot would incorrectly merge two table contexts.

### 3. The reader table is the top-level `f` created before `w()`

`a()` initializes a table and passes it through:

```luau
local f = {}
S,r,N = D:F(r,S,N,f)
h,S = D:w(f,S,h,N,r)
h,S = D:W(f,r,N,h,S)
...
S = D:go(N,f,S)
S = D:xo(f,N,S)
S = D:Fo(N,f,S)
```

`w()` clearly installs binary reader operations on this table, including:

```luau
f[0B1011] = nil
f[0B1010] = S.readi16
f[0B1011] = D.La
```

and `b()` installs:

```luau
N[0x8] = f.readu8
```

`W()` subsequently installs:

```luau
f[0x10] = _.readu16
f[0x13] = _.readu32
```

So `Fo()`'s `_` is the same long-lived reader/state table, not a newly-created independent parser object.

### 4. `Fo()` is the decisive parser-entry point

The relevant code is structurally:

```luau
f = (function()
    local D = _[50]()
    local f = _[0B110](D)
    local N = 0x74
    ...
    _[0B11] = _[0X3] + D
    ...
    _[0B11010](f, 0x0, _[0X25], _[0B11], D)
    ...
end)
```

Therefore slot 6 is a metadata/header decoder or constructor whose return value is passed as the first argument (`meta`) to the second-stage parser.

### 5. Slot 50 is a binary reader too

`Fo()` first executes `reader[50]()`. Elsewhere `reader[50]` is the cursor-consuming primitive used to obtain a variable-length/start value. The subsequent expression:

```luau
reader[0x3] = reader[0x3] + start
```

shows that `start` is interpreted as a byte/record offset relative to the reader cursor.

This makes the most likely high-level sequence:

```text
reader state
   |
   +-- slot 50 --> start / length / header-derived offset
   |
   +-- slot 6  --> meta/header object
   |
   +-- cursor += start
   |
   +-- slot 26 --> parse binary records using meta + buffer + cursor
```

This is a stronger model than assuming slot 6 is a normal byte reader.

## Important separation from `ig()`

`ig()` contains:

```luau
(f)[0X6] = S
```

but it is called from the function-construction routine inside `aa()`:

```luau
D:ig(j,J,K,p,Y,v,b)
```

and writes several adjacent frame fields (`f[4]`, `f[7]`, `f[1]`, `f[5]`, `f[0x6]`). This is a different table context from the `Fo()` reader. It should not be used as the implementation of `reader[6]` without an alias proof.

## Current conclusion

The exact body behind `reader[6]` is still indirectly installed/aliased and has not been safely recovered from the flattened Luraph table construction. However, its role is now constrained:

- it is called with the result of `reader[50]()`;
- its return value becomes `meta`;
- `meta` is passed directly into parser slot `26`;
- parser slot `26` also receives `0`, the decoded buffer, the updated cursor, and the VM/environment object.

So the next high-value target is **the constructor/alias chain that populates reader slot 6**, followed immediately by the first byte/tag read performed by parser slot 26.

## Next pass

1. Trace every function returning the table later used as `Fo()`'s `_`.
2. Track aliases of that table through `go`, `xo`, `Fo`, and their callees.
3. Resolve the exact value/function assigned to key `6` through those aliases rather than source-wide textual matching alone.
4. Once slot 6 is resolved, trace slot 26's first read and derive the binary record tag/length format.
5. Build `record_tag_map.md` only after the first tag dispatch is proven from source.

Original `loadervodo.luau.txt` remains untouched.