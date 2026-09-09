# Vodo Luraph v14.8 deobfuscation workspace

Target: `loadervodo.luau.txt`

This branch is dedicated to static analysis and controlled VM-state investigation of the Luraph v14.8 protected loader.

## Current findings

- The protected file is Luraph Obfuscator v14.8.
- The virtual machine contains an explicit `loadstring` opcode/assignment.
- The current emulator reaches a state where `0x2AC4 + 0x18AF` can become `nil + number`.
- `H` initializes state slot `0x2AC4`.
- `E` initializes state slot `0x18AF` and is reached through `G` in the analyzed control flow.
- `l` can fall back to `D:f` when `N[0x3753]` is absent; this is the main state transition being investigated.

## Safety

Captured/generated payloads must not be executed. Analysis should remain static or use a sandbox that prevents network/external side effects.
