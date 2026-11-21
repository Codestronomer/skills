---
name: ghidra-macos-arm64-headless
description: "Use when Ghidra decompilation fails silently on macOS ARM64, or when setting up headless Ghidra scripting — build the missing decompile native from source and script decompilation by address"
metadata:
  origin: auto-extracted
---

# Ghidra macOS ARM64: Build Missing Decompiler Native + Headless Scripting

**Extracted:** 2026-11-21
**Context:** Ghidra 12.x releases may omit the macOS ARM64 decompiler native binary, causing all decompilation to fail silently (empty error message, `decompileCompleted() == false`)

## Problem
Ghidra 12.x releases sometimes ship only `linux_x86_64` and `win_x86_64` natives in
`Ghidra/Features/Decompiler/os/`. On macOS ARM64, the decompiler engine silently fails
with an empty error message — analysis works (function discovery, disassembly) but
`DecompInterface.decompileFunction()` always returns `decompileCompleted() == false`.

## Solution

### 1. Verify the native is missing
```bash
ls $GHIDRA_INSTALL_DIR/Ghidra/Features/Decompiler/os/
# If macos_arm64/ or mac_arm_64/ is absent, the native is missing
```

### 2. Build from source (Ghidra repo clone)
```bash
cd ghidra/Ghidra/Features/Decompiler/src/decompile/cpp
# Patch the Makefile for ARM64:
sed -i '' 's/  ARCH_TYPE=-arch x86_64/  ARCH_TYPE=-arch arm64/' Makefile
sed -i '' 's/  OSDIR=mac_x86_64/  OSDIR=macos_arm64/' Makefile
sed -i '' 's/  ADDITIONAL_FLAGS=-mmacosx-version-min=10.6 -w/  ADDITIONAL_FLAGS=-w/' Makefile
sed -i '' 's/^CXX=g++ -std=c++11/CXX=g++ -std=c++17/' Makefile
make ghidra_opt -j8   # only builds the optimized decompiler (no BFD dependency)
```

### 3. Install into the release
```bash
mkdir -p $GHIDRA_INSTALL_DIR/Ghidra/Features/Decompiler/os/macos_arm64
cp ghidra_opt $GHIDRA_INSTALL_DIR/Ghidra/Features/Decompiler/os/macos_arm64/decompile
chmod +x $GHIDRA_INSTALL_DIR/Ghidra/Features/Decompiler/os/macos_arm64/decompile
# Also create mac_arm_64/ (Ghidra checks both names)
```

### 4. Headless decompilation by address
When decompiling functions in a stripped binary, export symbols may resolve to
GOT trampolines instead of real implementations. Decompile by address instead:
- Ghidra loads ELF at base `0x100000` — add `0x100000` to file offsets
- Use `CreateFunctionCmd` if no function exists at the target address
- Parse `DT_RELA` relocations to resolve runtime-filled function pointer tables

```java
// Ghidra headless post-script pattern
Address addr = currentProgram.getAddressFactory().getDefaultAddressSpace()
    .getAddress(Long.parseLong(hexOffset, 16));
Function fn = currentProgram.getFunctionManager().getFunctionAt(addr);
if (fn == null) {
    new CreateFunctionCmd(addr).applyTo(currentProgram, monitor);
    fn = currentProgram.getFunctionManager().getFunctionAt(addr);
}
DecompileResults res = decompiler.decompileFunction(fn, 300, monitor);
```

### 5. GOT trampoline detection
Exported symbols in stripped ARM64 binaries may be GOT trampolines (indirect jumps
through the Global Offset Table) rather than real function bodies. Symptoms:
- `nm -D` shows the symbol at a valid address
- Ghidra decompiles it as a single `return (*(code*)PTR_xxx)()` call
- The real implementation is at the PLT target address

```bash
# Find the real implementation address via PLT target
objdump -d binary.so | grep 'bl.*<symbol_name@plt>'
# The PLT target address IS the real function — decompile that address instead
```

## When to Use
- Ghidra decompilation fails with empty error on macOS ARM64
- Setting up automated decompilation of stripped ARM64 ELF binaries
- Need to decompile functions resolved via GOT/PLT trampolines
