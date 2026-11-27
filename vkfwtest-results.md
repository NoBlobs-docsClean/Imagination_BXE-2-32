<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# `vkfwtest` Reference & Baseline Results

This document records reproducible reference runs of the clean-room Vulkan test suite
`vkfwtest` on the **Banana Pi F3 (SpacemiT K1, PowerVR B-Series BXE-2-32 MC1)** across
both the **manufacturer firmware** and the **clean-room firmware**, on mainline Linux
(`powervr` + Mesa `pvr`) and on the vendor stack (`6.6.63` + `libVK_IMG.so`).

> **Provenance:** All results below are **O** (black-box board observations using only the
> public Vulkan API, `dmesg`, and standard Linux sysfs/module interfaces).

---

## 1. Latest baseline (`vkfwtest 0.2`, `powervr 1.1.0` + `Mesa pvr-bxe-2-32`)

### Manufacturer open firmware v1.1 (`build 6976702 OS`) — **48 PASS / 0 FAIL / 2 SKIP**
- **Date:** 2026-11-27.
- **Stack:** Arch Linux RISC-V (`mmcblk0p10`), kernel `7.4.0-rc1-img` with updated `powervr.ko`
  reporting DRM UAPI **1.1.0** (`DRM_PVR_DEV_QUERY_FW_INFO_GET`, enabling `gs_rta_support`
  automatically without Mesa fallback patches), `Mesa 27.0.0-devel` (`pvr-bxe-2-32`, with
  `VK_SUBPASS_EXTERNAL` barriers and layered-`LOAD` clear fix), `vkfwtest 0.2` (`8932a2f5`).
- **Full battery (`vkfwtest -k all`):** **`48 PASS, 0 FAIL, 0 TIMEOUT, 0 LOST, 0 ERROR, 2 SKIP`**
  (`local-read` and `sparse` skipped; **0** `dmesg` faults/resets/lockups):
  - `render` 19/20 (`rta` PASS `427.1 ms`, `bigpb-layers` PASS `7525.1 ms`),
    `second` 8/8, `compute` 4/4, `transfer` 5/5, `texture` 2/2, `sync` 2/2,
    `memory` 1/2, `robust` 1/1, `power` 2/2, `stress` 4/4 (`stress` PASS `1605.9 ms`).
- **Repeatability:**
  - `bigpb-layers`: **4/4 PASS** (`7525–7544 ms`), 0 `dmesg` faults.
  - `stress`: **14/14 PASS** (1× battery + 3× paired + 10/10 dedicated consecutive runs,
    `1572–1809 ms`), 0 `dmesg` lockups/faults.

---

## 2. Previous baseline (`vkfwtest 0.2`, commit `8932a2f5`, Mainline `7.4.0-rc1-img` + Arch Linux RISC-V, pre-fix Mesa)

### Reproducibility metadata
- **Board:** Banana Pi F3, SpacemiT K1 SoC, PowerVR B-Series BXE-2-32 MC1 (`BVNC 36.29.52.182`).
- **`vkfwtest` version / commit:** `vkfwtest 0.2`, commit `8932a2f5337f969a9ada15496d2b3b43dc31e6e7`
  (`2026-11-27 10:45:16 -0300`, *"Add second-render, baseline, layered-PB, power and context-switch tests; read-back barrier"* — 57 total tests, 50 in `all`).
- **Date of runs:** 2026-11-27.
- **OS / Kernel:** Arch Linux RISC-V (`mmcblk0p10`), `Linux k1-archriscv 7.4.0-rc1-img #2 SMP PREEMPT_DYNAMIC Sun Nov 8 14:27:13 -03 2026 riscv64`.
- **Mainline kernel modifications required on SpacemiT K1:**
  1. **SpacemiT K1 APMU power-domain support** (3-patch series):
     - `0001-dt-bindings-power-spacemit-Add-power-domain-controller.patch`
     - `0002-pmdomain-spacemit-Add-power-domain-driver.patch`
     - `0003-dts-riscv-spacemit-Add-power-domain-nodes.patch`
  2. **Device tree (`/boot/spacemit/7.4.0-rc1-img/k1-bananapi-f3.dtb`):** GPU node enabled
     with BXE-2-32 compatible string, power-domain phandle, and stable GPU clock (**819.2 MHz**).
  3. **Optional `powervr` context-switch module parameter** (`powervr-ctx-switch-param.patch`):
     adds `ctx_switch` module parameter to `drivers/gpu/drm/imagination/pvr_fw.c` (default `0`).
  4. **Driver loading:** `modprobe.blacklist=powervr` on kernel cmdline; loaded manually
     with `sudo modprobe powervr exp_hw_support=1`.
- **Mesa (`pvr` Vulkan driver):** `Mesa 27.0.0-devel` (`gs_rta_support` off for main
  baseline; `gs_rta_support` on for RTA-enabled check).
- **Firmwares compared:**
  - **Manufacturer open firmware:** `/usr/lib/firmware/powervr/rogue_36.29.52.182_v1.fw` —
    `dmesg`: `FW version v1.1 (build 6976702 OS)`.
  - **Clean-room firmware:** `/usr/lib/firmware/updates/powervr/rogue_36.29.52.182_v1.fw` —
    `dmesg`: `FW version v1.1 (build 1 OS)` (`522ec5a0`).
- **Invocation:**
  ```bash
  sudo modprobe -r powervr 2>/dev/null || true
  sudo dmesg -C
  sudo modprobe powervr exp_hw_support=1
  export VK_ICD_FILENAMES=<mesa-prefix>/share/vulkan/icd.d/powervr_mesa_icd.riscv64.json
  export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
  ./build/vkfwtest -k all
  ```

### Side-by-side results (`vkfwtest -k all`, `gs_rta_support` off)

| Group | Test | Manufacturer open FW (v1.1 build 6976702) | Clean-room FW (v1.1 build 1) | Notes |
|---|---|---|---|---|
| **render** | `multiframe` | **PASS** (521.5 ms) | **PASS** (551.8 ms) | 256×256, 8 frames |
| **render** | `pre-ta` | **PASS** (1084.6 ms) | **PASS** (1085.2 ms) | 512×512, 131072 tris, 4 frames |
| **render** | `bigpb` | **PASS** (3308.7 ms) | **PASS** (3691.0 ms) | 1024×1024 + D32, 4.19M tris, 2 frames (OOM + PR) |
| **render** | `bigpb-layers` | **PASS** (7495.6 ms) | **PASS** (7859.6 ms) | 1024×1024 × 4 layers + D32, 4.19M tris |
| **render** | `input-attach` | **PASS** (174.9 ms) | **PASS** (197.9 ms) | 64×64, 2 subpasses (colour + D32 inputs), 4 frames |
| **render** | `rta` | **PASS** (428.6 ms) | **PASS** (555.1 ms) | 64×64 × 6 layers, colour+D32, layers 1+5, D24S8, 3D-as-2D, multiview 6 |
| **render** | `zs-load` | **PASS** (180.3 ms) | **PASS** (210.7 ms) | 64×64 D24S8 across passes |
| **render** | `msaa` | **PASS** (183.3 ms) | **PASS** (212.3 ms) | 64×64 2×/4× + depth resolve + diagonal edge |
| **render** | `mrt` | **PASS** (182.3 ms) | **PASS** (203.8 ms) | 96×64, 4 colour attachments |
| **render** | `sizes` | **PASS** (14844.2 ms) | **PASS** (14893.6 ms) | 11 sizes 1×1..16384×16 + partial renderArea |
| **render** | `vertex` | **PASS** (226.8 ms) | **PASS** (254.3 ms) | 16/32-bit indexed, instanced, points, lines |
| **render** | `indirect` | **PASS** (194.1 ms) | **PASS** (214.9 ms) | draw / indexed / multi-draw + dispatch indirect |
| **render** | `queries` | **PASS** (181.4 ms) | **PASS** (202.1 ms) | occlusion precise `960 / 0 / 0` (host + GPU copy) |
| **render** | `formats` | **PASS** (211.4 ms) | **PASS** (251.1 ms) | 7 cases pass |
| **render** | `secondary` | **PASS** (144.8 ms) | **PASS** (164.4 ms) | 3 secondary command buffers |
| **render** | `multiview` | **PASS** (150.3 ms) | **PASS** (177.1 ms) | 64×64 × 2 views (`gl_ViewIndex`) |
| **render** | `raster` | **PASS** (245.0 ms) | **PASS** (293.4 ms) | neg-vp, strip+restart, fan, line, bias, wide-line, large-pt, mask, ss+a2c, no-att, dyn-render |
| **render** | `load-store-none` | **PASS** (166.4 ms) | **PASS** (190.9 ms) | `STORE_OP_NONE` + `LOAD_OP_NONE` |
| **render** | `local-read` | **SKIP** (121.4 ms) | **SKIP** (135.1 ms) | Needs Vulkan 1.4 / `VK_KHR_dynamic_rendering_local_read` |
| **render** | `vertex-ext` | **PASS** (180.3 ms) | **PASS** (202.5 ms) | 8-bit indices, divisor 4, divisor 0 |
| **second** | `second-same` | **FAIL** (143.1 ms) | **FAIL** (162.5 ms) | Identical Mesa `pvr` bug (missing implicit barrier) |
| **second** | `second-resubmit` | **FAIL** (141.5 ms) | **FAIL** (162.7 ms) | Identical Mesa `pvr` bug |
| **second** | `second-newfb` | **FAIL** (148.0 ms) | **FAIL** (165.7 ms) | Identical Mesa `pvr` bug |
| **second** | `second-newall` | **FAIL** (154.7 ms) | **FAIL** (177.3 ms) | Identical Mesa `pvr` bug |
| **second** | `second-compute` | **FAIL** (148.3 ms) | **FAIL** (166.1 ms) | Identical Mesa `pvr` bug |
| **second** | `second-xfer` | **FAIL** (144.9 ms) | **FAIL** (164.6 ms) | Identical Mesa `pvr` bug |
| **second** | `second-split` | **PASS** (146.0 ms) | **PASS** (164.2 ms) | Read back in separate submission |
| **second** | `second-barrier` | **PASS** (145.0 ms) | **PASS** (163.8 ms) | Explicit pipeline barrier before copy |
| **compute** | `pre-cdm` | **PASS** (134.1 ms) | **PASS** (150.2 ms) | 64×64, 16 LCG iters, 4 dispatches |
| **compute** | `resources` | **PASS** (162.2 ms) | **PASS** (185.1 ms) | UBO, dyn SSBO, storage img, texel bufs, atomics, shared mem, frag img, push desc |
| **compute** | `compute-dims` | **PASS** (165.7 ms) | **PASS** (180.6 ms) | 3D `dispatchBase` + max workgroup 128 reduction |
| **compute** | `subgroups` | **PASS** (132.0 ms) | **PASS** (147.9 ms) | `subgroupSize=32`, elect + ballot + subgroupAdd |
| **transfer** | `xfer` | **PASS** (590.5 ms) | **PASS** (624.0 ms) | 137×91 NPOT + 256×256: upload, offset copy, mirrored blit, MSAA resolve |
| **transfer** | `buffers` | **PASS** (140.5 ms) | **PASS** (156.3 ms) | `fillBuffer`, `updateBuffer`, `copyBuffer` |
| **transfer** | `ds-copy` | **PASS** (132.0 ms) | **PASS** (147.9 ms) | `D32_SFLOAT` + stencil clear/copy |
| **transfer** | `blit-mips` | **PASS** (144.5 ms) | **PASS** (166.5 ms) | `LINEAR` blits + 7-level mip chain |
| **transfer** | `image-copy` | **PASS** (141.1 ms) | **PASS** (172.2 ms) | 3D sub-volume, 2D array, `R32_UINT` → `RGBA8` |
| **texture** | `texture` | **PASS** (198.8 ms) | **PASS** (245.5 ms) | 37×23 NPOT, 4 address modes, 2 borders, linear, 6 mips |
| **texture** | `texture2` | **PASS** (233.8 ms) | **PASS** (268.2 ms) | cube+arr, 3D, array, shadow, gather, int/float, unnorm, ETC2, aniso |
| **sync** | `sync` | **PASS** (284.3 ms) | **PASS** (611.7 ms) | binary sem, timeline sem, events, 500 submits, `sync_file` |
| **sync** | `sync2` | **PASS** (131.7 ms) | **PASS** (152.3 ms) | `vkQueueSubmit2` + `vkCmdPipelineBarrier2` |
| **memory** | `memory` | **PASS** (361.3 ms) | **PASS** (402.3 ms) | 64 MiB alloc, 256 small allocs, 200 map/unmap cycles |
| **memory** | `sparse` | **SKIP** (121.9 ms) | **SKIP** (137.0 ms) | Sparse buffer residency not supported in Mesa `pvr` |
| **robust** | `robust` | **PASS** (243.9 ms) | **PASS** (268.3 ms) | OOB read `0x1000`, OOB write discarded, no fault |
| **power** | `idle-resume` | **PASS** (720.5 ms) | **PASS** (747.9 ms) | render → idle 550 ms → render on same device |
| **power** | `suspend-after` | **FAIL** (3354.5 ms) | **PASS** (754.4 ms) | Clean FW suspends in 100 ms; manufacturer FW stayed active > 3050 ms |
| **stress** | `stress` | **PASS** (1599.2 ms) | **FAIL** (61297.0 ms) | Clean FW: concurrent `multiframe` frame 12 fail + 3 context resets (overrun / page fault `0x28000`) |
| **stress** | `soak` | **PASS** (12981.7 ms) | **PASS** (13317.7 ms) | 120× multiframe, 120× pre-cdm, 20× pre-ta, 10× xfer |
| **stress** | `churn` | **PASS** (5374.7 ms) | **PASS** (6050.0 ms) | 20 devices created/destroyed |
| **stress** | `ctx-switch` | **PASS** (305.2 ms) | **PASS** (343.5 ms) | 2 devices queued together across 4 rounds |

### Totals (`vkfwtest -k all`, `gs_rta_support` off)

| Metric | Manufacturer open FW (v1.1 build 6976702) | Clean-room FW (v1.1 build 1) |
|---|---|---|
| **PASS** | **41** | **41** |
| **FAIL** | **7** (6× `second-*` + `suspend-after`) | **7** (6× `second-*` + `stress`) |
| **TIMEOUT / LOST / ERROR** | **0** | **0** |
| **SKIP** | **2** (`local-read`, `sparse`) | **2** (`local-read`, `sparse`) |
| **Kernel `dmesg` faults / resets** | **0** | **3** (all during `stress`: ctx 12 overrun, ctx 20 page fault `0x28000` + overrun) |

### RTA-enabled Mesa check (`gs_rta_support` on)
- **Manufacturer open FW (v1.1 build 6976702):** `rta` **FAIL** (`433.0 ms`, `subset layer 5: 3072 of 4096 pixels wrong`, 0 faults).
- **Clean-room FW (v1.1 build 1):** `rta` **FAIL** (`531.9 ms`, `subset layer 5: 3072 of 4096 pixels wrong`, 0 faults — identical pixel values to manufacturer FW, confirming a Mesa `pvr` bug on layered `LOAD` + `vkCmdClearAttachments`).

---

## 3. Historical vendor stack reference (`vkfwtest` commit `db6f34df`, Vendor `6.6.63` + `libVK_IMG.so`)

To validate `vkfwtest` itself against a conformant Vulkan 1.3 driver on the exact same
BXE-2-32 hardware, an earlier run was performed on the eMMC vendor stack:
- **Kernel:** `Linux k1 6.6.63 #2.2.9.2 SMP PREEMPT Tue Jan 27 21:48:13 UTC 2026 riscv64` (`pvr 24.2.6603887`).
- **Vulkan ICD:** `/etc/vulkan/icd.d/powervr_icd.json` → `libVK_IMG.so` (Vulkan `1.3.277`).
- **Firmware:** vendor closed firmware `rgx.fw.36.29.52.182` + `rgx.sh.36.29.52.182`.
- **`vkfwtest` commit:** `db6f34df3367450c0527971c36943961f00a5fe6` (39 tests in `all`), built with GCC `13.2.0` (`-std=gnu2x`).
- **Result (`sudo ./build/vkfwtest -k all`):** **`38 pass, 0 fail, 0 timeout, 0 lost, 0 error, 1 skip`** (`local-read`), **`0` kernel errors in `dmesg`** (`render 18/19`, `compute 4/4`, `transfer 5/5`, `texture 2/2`, `sync 2/2`, `memory 2/2`, `robust 1/1`, `power 1/1`, `stress 3/3`).
- **Preemption (`sudo ./build/vkfwtest -k prio-3d prio-ta prio-cdm`):** **`3/3 PASS`**, `0` kernel errors.
