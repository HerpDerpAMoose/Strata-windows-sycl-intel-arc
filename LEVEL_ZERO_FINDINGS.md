# Level Zero on this machine: diagnosis and findings

Machine: Arc Pro B70 32 GB, Windows 11 24H2, conda `strata-sycl-261`
(`dpcpp_win-64` **2026.1.1**), engine 0.1.43. Everything below is measured on this box, 2026-10-11/12.

---

## TL;DR for the next person

1. **Level Zero on this card was never broken.** The driver was fine; the *driver version* was the
   problem, and the fix was a graphics-driver update to the generic Arc line (`32.0.101.9034`).
   The Arc Pro line's newest (`32.0.101.8976`) does **not** expose Level Zero through any SYCL
   adapter at all.
2. **The Level Zero **v1** adapter is much better than v2 here**: decode +24%, graph capture 34x
   faster. This contradicts PR #1940's reason for pinning v2 on.
3. **A missing DLL list silently hid all of this.** `ur_adapter_level_zero.dll` imports `UMF.dll`
   and `VCRUNTIME140*`, which `sycl/CMakeLists.txt` never stages beside `strata.exe`, so the UR
   loader fell back to OpenCL with no diagnostic. This is the bug most worth reporting upstream.
4. **Level Zero is faster but not yet safe as a default**: it stalls in the device-plan verify
   window on the server request path. It is opt-in (`STRATA_SYCL_LEVEL_ZERO=1`) until that is fixed.

---

## How we got here, in order

### 1. The OpenCL fallback was built on a false premise

The port ran on OpenCL because `ONEAPI_DEVICE_SELECTOR` was never satisfied: `sycl-ls` enumerated
only `[opencl:gpu]`. The assumption was "Level Zero is broken on this card." It was not.

### 2. The driver is healthy; the loader above it was not

Bypassing SYCL entirely, through the v1 loader the driver shipped
(`...iigd_dch.inf...\ze_loader.dll`, 1.2.3), with no environment set up at all:

```
zeInit(0)             -> 0x0
drivers               -> 1
zeDeviceGet(count)    -> 0x0, devices=1
zeDriverGetProperties -> 0x0, driverVersion 17013007 (1.15.39183+7)
zeDeviceGetProperties -> 0x0
    type                  -> TYPE_GPU (1)
    vendorId / deviceId   -> 0x8086 / 0xe223          (Arc Pro B70, BMG-G31)
    maxMemAllocSize       -> 31.2 GiB
```

The modern loader (1.32.0, conda) returned `ZE_RESULT_ERROR_UNINITIALIZED` in *every*
configuration: bare, with `ZE_ENABLE_LOADER_DRIVER_SEARCH=1`, with the ICD directory on `PATH`, and
with the ICD already beside it. There is no `HKLM\SOFTWARE\Khronos\LevelZero` key on this box (OpenCL
and Vulkan are registered; Level Zero is not).

Registering the ICD by hand under `HKCU\SOFTWARE\Khronos\LevelZero\ICDs` turned the silent failure
into a straight `0xC0000005` crash - which proved it was never "the loader cannot find the ICD";
once it found it, it crashed against it. That registration was removed.

The standalone oneAPI 2026.0.0 installer was downloaded, hash-verified and installed. It also ships
**no `ze_loader.dll` at all** and behaves identically, so the toolchain packaging was not the answer
either.

### 3. The graphics driver was the answer

Two driver lines exist for this card:

| line | newest | exposes Level Zero? |
|---|---|---|
| Intel Arc **Pro** | 32.0.101.8976 (Q3.26) | **no** |
| Intel Arc (generic) | 32.0.101.9034 | **yes** |

Updating to the generic line's `32.0.101.9034` produced, for the first time:

```
[level_zero:gpu][level_zero:0] Intel(R) oneAPI Unified Runtime over Level-Zero V2,
    Intel(R) Arc(TM) Pro B70 Graphics 20.2.0 [1.17.39500+2]
    Aspects: ... ext_intel_free_memory ... ext_oneapi_graph ext_oneapi_limited_graph ...
```

Both things that had forced the OpenCL workarounds are present: **the free-VRAM query** and
**command graphs**. Installer: `%TEMP%\opencode\gfx_win_101.9034.exe`, SHA512 verified against
Intel's published digest.

### 4. A second, separate bug: the adapter's dependencies were never staged

With Level Zero available, the engine *still* only loaded `ur_adapter_opencl.dll`. The PE import
table of the Level Zero adapter explains why:

```
imports: UMF.dll, KERNEL32.dll, MSVCP140.dll, libmmd.dll, ADVAPI32.dll,
         VCRUNTIME140.dll, VCRUNTIME140_1.dll, api-ms-win-crt-*.dll, ...
```

`UMF.dll` and the `VCRUNTIME140*` set are not in the DLL-copy block in `sycl/CMakeLists.txt`
(which copies `sycl9.dll libmmd.dll ur_loader.dll ur_win_proxy_loader.dll ur_adapter_*.dll
OpenCL.dll`). The adapter therefore fails to load, the UR loader reports only OpenCL, and
`ONEAPI_DEVICE_SELECTOR=level_zero:0` kills the process with `0xC0000409` before `main` prints
anything.

**Confirmed workaround: put conda's `Library\bin` first on `PATH`.** The proper fix is to extend the
copy list - and the *silence* is the part worth reporting upstream, because it masks every Level Zero
adapter failure as "there is no Level Zero here."

---

## Measured, driver 9034

All runs: Arc Pro B70, coder-IQ1_M, `--spec 4 --spec-min-p 0.5`, `--mtp`, MTP draft head loaded,
2047-token prompt, greedy.

| configuration | decode | prefill | graph capture | stall? |
|---|---|---|---|---|
| OpenCL, `STRATA_VERIFY_EAGER=1` | 66.6 tok/s | 81.1 | skipped | no |
| LZ **v2** adapter | 87.9 tok/s | 64.5 | 9,495 ms | no |
| LZ **v1** adapter, 128 tok | 108.8 tok/s | 81.1 | 276 ms | no |
| LZ **v1** adapter, 512 tok, MTP | **74.5 tok/s** | **121.4 tok/s** | — | no |
| LZ, device plan **OFF** | 0.95 tok/s | 106.5 tok/s | — | no |
| LZ via the **server** | — | — | — | **yes** |

**The v1 Level Zero adapter beats v2 decisively**: decode +24% and graph capture 34x faster.
#1940 pins `SYCL_UR_USE_LEVEL_ZERO_V2=1` for "graph record and replay, no event leak per graph
finalize (intel/llvm#23073)"; on this card, with this workload, that trade is clearly wrong.

**The device plan is required**: with it off, decode is 0.95 tok/s - 100x slower - because the
window's host handshakes become synchronous. It is also where the stall lives.

**Prefill is +50% on Level Zero** (121.4 vs 81.1 tok/s) with oneMKL refusing in both cases, so the
plain-SYCL fallback GEMM is itself faster on Level Zero than on OpenCL.

---

## The stall, and exactly what it is not

The message, whichever way it is reached:

```
verify window: waiting for the GPU to finish the window (flags A/B/M raised) for 60 s
verify window: 4 tokens at position 84, host at layer step 1; the GPU rang 0
              flags: served 0, plan (A) 0, copies (B) 0
wrote the thread stacks to strata-stall-<pid>.dmp
```

Ruled out by direct measurement:

- **Not the launcher environment.** The same arguments through `strata-sycl.bat` with its full
  environment complete 512 decode tokens cleanly at 74.5 tok/s.
- **Not prompt length or token count.** A 2047-token prompt with 512 decode tokens is stable; the
  server stalls on a 4-token window at KV position ~84.
- **Not the driver-search variable or a stale ICD** (the remaining launcher difference; conda's
  `Library\bin` holds an old `ze_intel_gpu64.dll` at 32.0.101.8976).
- **Not `--mtp`** - removing it did not stop the server stall.

So the trigger is in what the **server request path** does differently: prompt-prefix reuse across
requests, the serve protocol, or the accumulation of KV position over several requests. That is
where to instrument next - which of flags A / B / M the GPU should raise, and whether it is
written-but-invisible (a memory-ordering problem, Level Zero specific) or never written at all (a plan
problem, which would not be).

---

## oneMKL: closed

oneMKL refuses with *"The oneAPI Math Kernel Library (oneMKL) Interfaces Project does not support
this API"* against **both** the PyPI runtime (2026.1.0) and the standalone oneAPI 2026.0 runtime,
with identical timings - so the version is not the reason. The engine's expert GEMMs are **BF16**
(`sycl/src/prefill/gemml.dp.cpp:1031`), which is the likely unsupported type. Still a hypothesis;
the cheap way to settle it is to call oneMKL with FP16 operands, since `Gemm::f16` exists on the
same file.

---

## What the machine needs for Level Zero

```powershell
# 1. conda's Library\bin FIRST on PATH (the UR adapters' dependencies live there)
$c = "$env:USERPROFILE\miniforge3\envs\strata-sycl-261\Library\bin"
$env:PATH = "$c;$env:PATH"

# 2..4, and STRATA_VERIFY_EAGER must be UNSET
$env:ONEAPI_DEVICE_SELECTOR    = "level_zero:0"
$env:SYCL_UR_USE_LEVEL_ZERO_V2 = "0"      # v1, not v2: +24% decode, 34x faster graph capture
$env:ZES_ENABLE_SYSMAN          = "1"      # makes ext_intel_free_memory answer

# 5. a graphics driver on the GENERIC Arc line, 32.0.101.9034 or newer.
#    The Arc Pro line's newest (8976) does not work.
```

`sycl/serve/strata-sycl.bat` does all of this behind `STRATA_SYCL_LEVEL_ZERO=1`, off by default.

---

## What this retires, once the stall is fixed

| workaround | retired by |
|---|---|
| `STRATA_VERIFY_EAGER` + the commit replay | `ext_oneapi_graph` now exists |
| the plain-SYCL fallback GEMM | oneMKL, if BF16 turns out not to be the blocker |
| the DXGI live free-VRAM figure | `ext_intel_free_memory` now exists; `device_free_bytes()` should then prefer the backend's answer over DXGI |
