# VM state map — first pass

## Relevant functions

### `a`
Top-level initializer. Calls `F`, then `w`, `W`, and `k` in sequence.

### `W`
Runs the `j` state machine beginning at `S = 0x4F`. Later initializes `f[0x10]` and conditionally calls `H` or `l`.

### `j`
For `f <= 0x4F`, calls `D`; otherwise calls `X`.

### `D`
Creates `A[14] = {}` and either computes `D:v(...)` or reuses `D:K(...)` depending on `_[0x58B9]`.

### `H`
Sets `f[0x10] = _.readu16`. If `N[10948]` is absent, computes and stores `N[0x2AC4]`; otherwise copies `N[10948]`.

### `l`
Sets `A[0x11] = _[D.g]`, `A[0x12] = D._`. If `N[0x3753]` is present, uses `N[0x3753]`; otherwise calls `D:f(N,f)`.

### `f`
Computes a value using `f[0x2AC4] + f[0x18AF]`, then stores the result in `f[14163]`.

### `E`
Explicitly computes and stores `f[0x18AF]`. Its call site is through `G` when `f[13968]` is absent.

## Main unresolved branch

The observed crash occurs at the arithmetic expression involving slots `0x2AC4` and `0x18AF`. The next analysis target is determining the exact `W -> H/l -> D:f` branch and why `E` has not populated `0x18AF` on that path. Do not patch either slot to force execution; derive the original state transition first.
