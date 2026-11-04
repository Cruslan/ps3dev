# PS3DEV Modular Patchsets

This directory contains standalone, portable patchsets maintained by Cruslan for modern PlayStation 3 homebrew development. They fix critical hardware hangs, compiler register clobber bugs in GCC 13+, and memory alignment faults on the Cell Broadband Engine.

All patches are designed to be cleanly applicable via `git apply <patch>` or GNU `patch -p1` on top of upstream official repositories (`ps3dev/PSL1GHT` and `ps3dev/ps3libraries`).

---

## Patch Index

### 1. `patches/PSL1GHT/` (Target: https://github.com/ps3dev/PSL1GHT)

* **`0001-librsx-wrap-user-callback-in-__get_opd32.patch`**:
  * **Root Cause**: In PowerPC 64-bit ELF ABI, C function pointers refer to 24-byte 64-bit Procedure Descriptors (OPD). However, `rsxContextCallback` in `rsx_function_macros.h` uses 32-bit word loads (`lwz 0, 0(%3); lwz 2, 4(%3)`) expecting a 32-bit OPD descriptor layout.
  * **Fix**: Wraps user-supplied callback handlers in `__get_opd32` to guarantee valid 32-bit entry addresses and TOC pointers when the command buffer wraps.

* **`0002-librsx-cgcomp-fix-rsx-gpu-hangs-and-pointer-overflow.patch`**:
  * **Root Cause 1**: Pointer scaling overflow in `rsxSetupContextData`. Because `addr` is typed as `const u32*`, standard C pointer arithmetic (`addr + alignedSize`) scales by `sizeof(u32)` (4x byte count), placing `context->end` 4x beyond the allocated boundary and defeating wrap detection.
  * **Root Cause 2**: Fragment programs with texture sampling require the `allowTex` (bit 15 / `0x8000`) control bit set in `m_nFPControl` and `LoadFragmentProgramLocation`. Missing bit 15 caused RSX GPU hangs on hardware.
  * **Root Cause 3**: Second burst in `SetVertexProgramConstants` erroneously read from `value` instead of `&value[16]`.
  * **Root Cause 4**: `SetTimeStamp` wrote outside packet bounds (`currenp[2]` instead of `[1]`).
  * **Root Cause 5**: `rsxContextCallback` lacked volatile register clobbers in inline assembly, leading GCC 13+ to corrupt registers across indirect calls.
  * **Root Cause 6**: Added missing H.264 codec defines in `vdec.h`.

* **`0003-librsx-buffer-reset-spin-timeout-and-sync.patch`**:
  * **Root Cause**: `ResetCommandBuffer` spun infinitely in a `while(ctrl->get != offset)` loop without a timeout or memory barrier when RSX experienced FIFO desync.
  * **Fix**: Implements a 100,000-iteration bounded spin timeout limit and an explicit `__sync()` memory barrier before resetting `ctrl->put`.

---

### 2. `patches/ps3libraries/` (Target: https://github.com/ps3dev/ps3libraries)

* **`0001-tiny3d-enforce-mstrict-align.patch`**:
  * **Root Cause**: On Cell PPE (64-bit PowerPC architecture), GCC 13 with `-O2` optimizes paired 32-bit float writes into single 64-bit indexed store instructions (`stdx`). In Tiny3D (`tiny3d.c`), vertex structures are 12-byte float triplets (`x, y, z`), causing odd-numbered vertices to land on 4-byte boundaries that are NOT 8-byte aligned. Because `rsx_vertex` is mapped directly to RSX I/O memory, unaligned `stdx` stores trigger an unrecoverable **Hardware Alignment Exception (DSI / Machine Check)** on Cell PPE, immediately freezing the console with an unresponsive black screen.
  * **Fix**: Enforces `-mstrict-align` in `tiny3d/lib/Makefile` CFLAGS. This decomposes doubleword stores into safe, 32-bit `stw` instructions (`stdx` count drops from 7 to 0), completely eliminating the hardware alignment trap.

---

## How to Apply Manually

To apply any patch manually inside a checkout of the corresponding repository:

```bash
# In PSL1GHT checkout:
git apply /path/to/patches/PSL1GHT/*.patch

# In ps3libraries checkout:
git apply /path/to/patches/ps3libraries/*.patch
```
