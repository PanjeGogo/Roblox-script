# VM control-flow reconstruction — second pass

Target: `loadervodo.luau.txt` (Luraph v14.8), analyzed statically. This document records behavior that can be proven from the protected source; it does not claim recovery of the author's original identifiers.

## `a -> W` bootstrap

The top-level `a` function initializes the VM state through `F`, `w`, then `W`, followed by `k`, `m`, `t`, and later VM setup routines.

`W` contains two distinct state machines.

### Phase 1: `j` loop

```text
S = 0x4F
repeat
    r, S, N = j(S, A, f, N)
    if r == 0x5DAC then break
    if r == 17729 then continue
until break
```

`j` is straightforward:

```text
if S <= 0x4F then
    S = D(f, A, N)
    return 0x4541, S, N
else
    N = X(A, N)
    return 0x5DAC, S, N
end
```

`D` always creates `A[14] = {}`. It then either computes a value with `v` or reuses the cached value through `K`, depending on `A[0x58B9]`.

### Phase 2: `H/l` dispatch

After the first phase, `W` clears slots `f[0x10]`, `f[0x11]`, `f[18]`, and `f[19]`, then sets:

```text
S = 2
```

The exact control flow is:

```text
while true do
    if not (S > 2) then
        S = H(f, _, S, A)
        continue
    else
        if S == 0x79 then       -- decimal 121
            S = l(S, _, f, A)
            continue
        else
            f[0x13] = _.readu32
            break
        end
    end
end
```

Therefore `H` is entered while `S <= 2`. Once `H` returns a value greater than 2, `H` is no longer repeated. `l` is reached **only when the value returned by `H` is exactly `121`**. Otherwise the path installs `readu32` and exits this state machine.

## `H` state calculation

`H` first installs:

```text
f[0x10] = _.readu16
```

If `N[10948]` is absent, it computes a value from fields of `N` and `D.J`, then stores it as:

```text
N[0x2AC4] = A
```

If `N[10948]` already exists, it simply uses that value. It then returns `A`.

This means the branch condition for entering `l` is determined directly by the computed/cached `A` from `H`; it is not an unconditional call to `l`.

## `l` and the crash path

`l` initializes two fields:

```text
A[0x11] = _[D.g]
A[0x12] = D._
```

It then checks `N[0x3753]`:

```text
if N[0x3753] then
    f = N[0x3753]
else
    f = D:f(N, f)
end
```

The helper `D:f` evaluates an expression containing:

```text
f[0x2AC4] + f[0x18AF]
```

and stores the final result in `f[14163]`.

A separate helper `E` explicitly initializes `f[0x18AF]`, but its observed call site is through `G`, which only calls `E` when `f[13968]` is absent. Static inspection therefore does **not** justify replacing `f[0x18AF]` with a guessed constant. The nil arithmetic observed in the sandbox is consistent with reaching `D:f` before that slot has been initialized on the emulated path.

## Important conclusion

The earlier sandbox crash is now narrowed to a precise branch:

```text
W -> H returns 121 -> l -> N[0x3753] is nil -> D:f -> f[0x2AC4] + f[0x18AF]
```

This is a control-flow finding, not a claim that the protected script necessarily crashes in its intended Roblox/executor environment. The remaining task is to reconstruct the state-producing routines that determine whether `H` returns 121 and whether `N[0x3753]`/`f[0x18AF]` are populated on the intended path.

## Safety boundary

No captured payload is executed. The analysis is limited to static reconstruction of the VM/bootstrap layer and opcode semantics.
