<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# `vkfwtest` Reference Results (Vendor Stack vs. Mainline + Mesa `pvr`)

This document records two reproducible reference runs of the clean-room Vulkan test suite
`vkfwtest` on the **Banana Pi F3 (SpacemiT K1, PowerVR B-Series BXE-2-32 MC1)** using the
manufacturer's firmware binaries:

1. **Vendor stack** (SpacemiT vendor Linux `6.6.63` + Imagination vendor Vulkan driver
   `libVK_IMG.so` + vendor closed firmware `rgx.fw.36.29.52.182`): **38/38 supported tests
   PASS (`0` failures, `1` skip, `0` kernel errors)**.
2. **Mainline stack** (Linux `7.4.0-rc1-img` + upstream `powervr` DRM driver + Mesa `pvr`
   + manufacturer open firmware `rogue_36.29.52.182_v1.fw` v1.1 build `6976702 OS`):
   **18 PASS, 18 FAIL, 3 SKIP, `0` kernel/firmware faults**.

Together, these two runs establish that:
- `vkfwtest` itself is validated end-to-end against a conformant Vulkan 1.3 driver on the
  exact same BXE-2-32 silicon (`38/38 PASS`);
- the 18 pixel/output mismatches observed on the mainline run are **Mesa `pvr` driver
  limitations/bugs** (not firmware bugs and not `vkfwtest` bugs), providing an exact
  per-test baseline for evaluating any clean-room firmware on mainline Linux + Mesa `pvr`.

> **Provenance:** All results below are **O** (black-box board observations using only the
> public Vulkan API, `dmesg`, and standard Linux sysfs/module interfaces).

---

## 1. Reproducibility metadata

### Hardware & test suite
- **Board:** Banana Pi F3, SpacemiT K1 SoC, PowerVR B-Series BXE-2-32 MC1 (`BVNC 36.29.52.182`).
- **`vkfwtest` commit:** `db6f34df3367450c0527971c36943961f00a5fe6`
  (`2026-11-27 03:26:57 -0300`, *"Require only Vulkan 1.0; add optional 1.3/1.4 tests and --api"*;
  working tree clean except `docs/steps.md`).
- **Date of runs:** 2026-11-27.

### Run 1 — Vendor stack (eMMC)
- **Kernel:** `Linux k1 6.6.63 #2.2.9.2 SMP PREEMPT Tue Jan 27 21:48:13 UTC 2026 riscv64`
  (Bianbu vendor kernel, stock eMMC image, `pvr 24.2.6603887`).
- **Vulkan ICD:** `/etc/vulkan/icd.d/powervr_icd.json` → `libVK_IMG.so` (Vulkan `1.3.277`).
- **Firmware:** vendor closed firmware `rgx.fw.36.29.52.182` + `rgx.sh.36.29.52.182`.
- **Build on target:** GCC `13.2.0` (`Bianbu 13.2.0-23ubuntu4bb4`), using the pre-generated
  39 SPIR-V headers (`build/*.h`, compiled with `glslangValidator 15.2.0 --target-env vulkan1.1`),
  `-std=gnu2x` (substituted for `-std=gnu23` on GCC 13), Vulkan headers (`vulkan/vulkan.h`),
  and `/usr/lib/libvulkan.so`:
  ```bash
  sed -i 's/-std=gnu23/-std=gnu2x/' Makefile
  touch build/*.h
  make -j4 VK_CFLAGS="-Iinc" VK_LIBS="/usr/lib/libvulkan.so"
  ```
- **Invocation:**
  ```bash
  sudo dmesg -C
  sudo ./build/vkfwtest -k all
  ```

### Run 2 — Mainline stack + Mesa `pvr` + manufacturer open firmware (SD card)
- **Kernel:** `Linux k1-mainline 7.4.0-rc1-img #2 SMP PREEMPT_DYNAMIC Sun Nov 8 14:27:13 -03 2026 riscv64`.
- **Kernel command line:**
  ```text
  BOOT_IMAGE=/vmlinuz-7.4.0-rc1-img root=UUID=5d8f4431-e5d5-4827-beb5-0369ad52139d
  rootwait rootfstype=ext4 earlyprintk clk_ignore_unused pd_ignore_unused swiotlb=65536
  workqueue.default_affinity_scope=system console=ttyS0,115200 earlycon=sbi
  modprobe.blacklist=powervr
  ```
- **Mainline kernel modifications required on SpacemiT K1:**
  1. **SpacemiT K1 APMU power-domain support** (3-patch series for GPU power island control):
     - `0001-dt-bindings-power-spacemit-Add-power-domain-controller.patch`
     - `0002-pmdomain-spacemit-Add-power-domain-driver.patch`
     - `0003-dts-riscv-spacemit-Add-power-domain-nodes.patch`
  2. **Device tree (`/boot/spacemit/7.4.0-rc1-img/k1-bananapi-f3.dtb`):** GPU node enabled
     with BXE-2-32 compatible string, power-domain phandle, and stable GPU operating clock
     (**819.2 MHz**; 1 GHz exhibits intermittent compute/render errors at stock voltage).
  3. **Optional `powervr` context-switch module parameter** (`powervr-ctx-switch-param.patch`):
     adds `ctx_switch` module parameter to `drivers/gpu/drm/imagination/pvr_fw.c` to pass
     `ROGUE_FWIF_INICFG_OS_CTXSWITCH_{GEOM,FRAG,CDM}_EN` bits (default `0` in this run).
  4. **Driver loading:** `powervr` is blacklisted at boot (`modprobe.blacklist=powervr`) and
     loaded manually via `sudo modprobe powervr` after ensuring no override exists in
     `/lib/firmware/updates/powervr/`.
- **Firmware:** `/lib/firmware/powervr/rogue_36.29.52.182_v1.fw` — manufacturer open
  firmware (`dmesg`: `FW version v1.1 (build 6976702 OS)`).
- **Mesa (`pvr` Vulkan driver):** built from `/opt/src/mesa` at commit
  `f7c93a1352b578fbea6c249227d44df65b8f6a35` (`2026-10-21 04:32:29 +0000`,
  *"glsl/lower_instructions: Use ir_builder carry instead of local _carry helper"*).
- **Build on target:** GCC `14.2.0` (`Debian 14.2.0-19`), `glslangValidator 15.2.0`,
  unmodified `Makefile` (`-std=gnu23 -Wall -Wextra -Werror`):
  ```bash
  make -j4
  ```
- **Invocation (required env vars — without them Vulkan falls back to `llvmpipe`):**
  ```bash
  sudo modprobe -r powervr 2>/dev/null || true
  sudo dmesg -C
  sudo modprobe powervr
  export VK_ICD_FILENAMES=/opt/src/mesa/build/src/imagination/vulkan/powervr_mesa_devenv_icd.riscv64.json
  export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
  ./build/vkfwtest -k all
  ```

---

## 2. Side-by-side results (`vkfwtest -k all`)

| Group | Test | Vendor 6.6.63 + `libVK_IMG.so` | Mainline `7.4.0-rc1-img` + Mesa `pvr` + vendor open fw | Notes |
|---|---|---|---|---|
| **render** | `multiframe` | **PASS** (469.8 ms) | **FAIL** (362.3 ms) | Mainline: frame 0 passes; frame 1 reads frame 0 colour (`7e0116ff` vs `130c5fff`), 0 faults |
| **render** | `pre-ta` | **PASS** (882.2 ms) | **FAIL** (296.1 ms) | Mainline: frame 0 reads `00000000`, 0 faults |
| **render** | `bigpb` | **PASS** (2446.9 ms) | **FAIL** (1396.9 ms) | Mainline: frame 0 reads `00000000`, 0 faults |
| **render** | `input-attach` | **PASS** (158.0 ms) | **FAIL** (171.6 ms) | Mainline: frame 1 subpass 0 reads frame 0 colour, 0 faults |
| **render** | `rta` | **PASS** (130.2 ms) | **PASS** (188.3 ms) | 64×64 × 6 layers: layered clears (`gl_Layer` draw skipped on both) |
| **render** | `zs-load` | **PASS** (174.6 ms) | **FAIL** (171.9 ms) | Mainline: pass 2 (loaded depth) wrong pixels, 0 faults |
| **render** | `msaa` | **PASS** (159.8 ms) | **PASS** (170.5 ms) | 4× MSAA resolve + depth resolve + diagonal edge |
| **render** | `mrt` | **PASS** (262.7 ms) | **FAIL** (183.9 ms) | Mainline: frame 1 attachment 0 wrong pixels, 0 faults |
| **render** | `sizes` | **PASS** (12699.3 ms) | **FAIL** (170.5 ms) | Mainline: `33x17` mismatch, 0 faults |
| **render** | `vertex` | **PASS** (230.6 ms) | **FAIL** (182.2 ms) | Mainline: `frame 0 points` mismatch, 0 faults |
| **render** | `indirect` | **PASS** (253.0 ms) | **PASS** (207.9 ms) | draw / indexed / multi-draw indirect + dispatch indirect |
| **render** | `queries` | **PASS** (151.2 ms) | **PASS** (200.9 ms) | occlusion precise `960 / 0 / 0` (timestamps supported on vendor, skipped on Mesa `pvr`) |
| **render** | `formats` | **PASS** (303.8 ms) | **FAIL** (176.6 ms) | Mainline: `R8G8B8A8_SRGB` mismatch, 0 faults |
| **render** | `secondary` | **PASS** (122.1 ms) | **FAIL** (156.9 ms) | Mainline: frame 1 mismatch, 0 faults |
| **render** | `multiview` | **PASS** (135.4 ms) | **FAIL** (165.2 ms) | Mainline: frame 1 view 0 mismatch, 0 faults |
| **render** | `raster` | **PASS** (334.6 ms) | **FAIL** (163.7 ms) | Mainline: `strip + primitive restart` (`1152/4096` wrong), 0 faults |
| **render** | `load-store-none` | **PASS** (162.1 ms) | **SKIP** (133.1 ms) | Mesa `pvr` lacks `VK_KHR/EXT_load_store_op_none` |
| **render** | `local-read` | **SKIP** (87.3 ms) | **SKIP** (134.1 ms) | Needs Vulkan 1.4 / `VK_KHR_dynamic_rendering_local_read` |
| **render** | `vertex-ext` | **PASS** (175.3 ms) | **FAIL** (190.1 ms) | Mainline: `divisor 0` mismatch, 0 faults |
| **compute** | `pre-cdm` | **PASS** (115.8 ms) | **PASS** (147.5 ms) | 64 groups × 64, 16 LCG iters, 4 dispatches |
| **compute** | `resources` | **PASS** (193.3 ms) | **PASS** (188.1 ms) | UBO, dyn SSBO, storage img, texel bufs, atomics, shared mem, frag img, push desc |
| **compute** | `compute-dims` | **PASS** (150.8 ms) | **PASS** (180.2 ms) | 3D `dispatchBase` + max workgroup reduction (512 on vendor, 128 on Mesa `pvr`) |
| **compute** | `subgroups` | **PASS** (105.9 ms) | **PASS** (143.6 ms) | `subgroupSize=1` on vendor, `subgroupSize=32` on Mesa `pvr` |
| **transfer** | `xfer` | **PASS** (499.4 ms) | **PASS** (612.5 ms) | 137×91 NPOT + 256×256: upload, offset copy, mirrored blit, MSAA resolve |
| **transfer** | `buffers` | **PASS** (90.7 ms) | **PASS** (151.5 ms) | `fillBuffer`, `updateBuffer`, `copyBuffer` (incl. odd/1 B) |
| **transfer** | `ds-copy` | **PASS** (90.3 ms) | **PASS** (142.4 ms) | `D32_SFLOAT` + stencil clear/copy |
| **transfer** | `blit-mips` | **PASS** (101.2 ms) | **PASS** (157.5 ms) | `LINEAR` blits + 7-level mip chain |
| **transfer** | `image-copy` | **PASS** (93.7 ms) | **PASS** (161.7 ms) | 3D sub-volume, 2D array, `R32_UINT` → `RGBA8` |
| **texture** | `texture` | **PASS** (164.5 ms) | **FAIL** (152.7 ms) | Mainline: `frame 0 nearest repeat` reads `00000000`, 0 faults |
| **texture** | `texture2` | **PASS** (324.8 ms) | **PASS** (260.1 ms) | cube+arr, 3D, array, shadow, gather, int/float, unnorm, ETC2, aniso (ASTC on vendor) |
| **sync** | `sync` | **PASS** (176.2 ms) | **PASS** (392.6 ms) | binary sem, timeline sem, events, 500 small submits, `sync_file` fence |
| **sync** | `sync2` | **PASS** (91.7 ms) | **PASS** (145.3 ms) | `vkQueueSubmit2` + `vkCmdPipelineBarrier2` |
| **memory** | `memory` | **PASS** (145.0 ms) | **PASS** (380.3 ms) | 64 MiB alloc, 256 small allocs, 200 map/unmap cycles |
| **memory** | `sparse` | **PASS** (98.1 ms) | **SKIP** (132.1 ms) | Sparse buffer residency not supported in Mesa `pvr` |
| **robust** | `robust` | **PASS** (159.0 ms) | **PASS** (262.4 ms) | OOB read safe (`0x0` vendor / `0x1000` Mesa `pvr`), OOB write discarded, no fault |
| **power** | `idle-resume` | **PASS** (683.7 ms) | **FAIL** (160.1 ms) | Mainline: fails on frame 1 (same `multiframe` pattern), 0 faults |
| **stress** | `stress` | **PASS** (1200.8 ms) | **FAIL** (1121.3 ms) | Mainline: fails on `multiframe` frame 1, 0 faults |
| **stress** | `soak` | **PASS** (11065.7 ms) | **FAIL** (218.9 ms) | Mainline: fails on `multiframe` frame 1, 0 faults |
| **stress** | `churn` | **PASS** (3314.7 ms) | **FAIL** (280.3 ms) | Mainline: fails on device 1 frame 1, 0 faults |

### Totals

| Metric | Vendor 6.6.63 + `libVK_IMG.so` | Mainline `7.4.0-rc1-img` + Mesa `pvr` + vendor open fw |
|---|---|---|
| **PASS** | **38** | **18** |
| **FAIL** | **0** | **18** |
| **TIMEOUT / LOST / ERROR** | **0** | **0** |
| **SKIP** | **1** (`local-read`) | **3** (`load-store-none`, `local-read`, `sparse`) |
| **Kernel `dmesg` errors / faults / HWR** | **0** | **0** |

### Group breakdown
- **Vendor stack:** `render 18/19`, `compute 4/4`, `transfer 5/5`, `texture 2/2`, `sync 2/2`, `memory 2/2`, `robust 1/1`, `power 1/1`, `stress 3/3`.
- **Mainline + Mesa `pvr` + vendor open fw:** `render 4/19`, `compute 4/4`, `transfer 5/5`, `texture 1/2`, `sync 2/2`, `memory 1/2`, `robust 1/1`, `power 0/1`, `stress 0/3`.

---

## 3. Additional non-`all` tests (preemption & HWR, earlier run on same drivers)

- **Priority preemption (`prio-3d`, `prio-ta`, `prio-cdm`):**
  - **Vendor stack (`sudo`):** `3/3 PASS`, `0` kernel errors (probe averages 89–138 ms during 2.0–2.9 s low-priority jobs).
  - **Mainline + Mesa `pvr` (non-root):** `prio-cdm` **PASS** (3.6 s, `HIGH` priority unavailable without `CAP_SYS_NICE`); `prio-3d` and `prio-ta` failed calibration due to the same Mesa `pvr` render mismatch seen in `multiframe`/`pre-ta`.
- **HWR (`hwr-fault`, `hwr-lockup`):**
  - On **mainline + Mesa `pvr` + vendor open fw**: `hwr-lockup` triggered the firmware's 30 s watchdog and recovered (`pre-cdm` afterwards passed), but Mesa `pvr` signalled the fence without returning `VK_ERROR_DEVICE_LOST`.
  - On **vendor stack**: `hwr-fault` signalled without `VK_ERROR_DEVICE_LOST`; `hwr-lockup` triggered the vendor kernel watchdog (`pvr_device_wdg`) without returning `VK_ERROR_DEVICE_LOST` to the fence.

---

## 4. How to use this baseline when testing a clean-room firmware

When running `vkfwtest -k all` against a clean-room firmware on the same **mainline Linux + Mesa `pvr`** setup:
1. All **18 tests that pass** with the manufacturer open firmware (`rta`, `msaa`, `indirect`, `queries`, `pre-cdm`, `resources`, `compute-dims`, `subgroups`, `xfer`, `buffers`, `ds-copy`, `blit-mips`, `image-copy`, `texture2`, `sync`, `sync2`, `memory`, `robust`) **must PASS** with **`0` timeouts, `0` device-lost, `0` kernel faults/HWR in `dmesg`**.
2. The 18 tests that fail pixel comparison on Mesa `pvr` with the manufacturer open firmware must at minimum complete with **`0` timeouts, `0` device-lost, and `0` kernel faults/HWR** (until Mesa `pvr` itself is updated to fix the frame-1/render bugs).
