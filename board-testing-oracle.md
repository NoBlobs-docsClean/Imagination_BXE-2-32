<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# Board testing and acceptance oracle (SpacemiT K1)

This guide describes how the clean team can validate a from-scratch firmware image on the
**Banana Pi F3 (SpacemiT K1)** running the upstream mainline Linux `powervr` driver and
Mesa `pvr`, using **only** public interfaces and black-box observation (**P/O**).

---

## 1. Test platform and hardware safety

- **Target stack:** mainline Linux (`powervr` DRM driver + Mesa `pvr`, GPU at 819.2 MHz;
  1 GHz is known to show intermittent compute/render errors at stock voltage).
- **Firmware path:** `/lib/firmware/powervr/rogue_36.29.52.182_v1.fw`. Always keep a
  verified fallback copy before swapping the image, and run `sync` before reloading the
  module (`sudo modprobe -r powervr && sudo modprobe powervr`).
  **Warning (`updates/` precedence):** Linux `request_firmware()` searches
  `/lib/firmware/updates/` (and `/lib/firmware/updates/$(uname -r)/`) *before*
  `/lib/firmware/`. Make sure no stale copy exists under `/lib/firmware/updates/powervr/`
  (or remove the `updates/` override), otherwise the kernel will silently load the file
  from `updates/` and ignore changes written to `/lib/firmware/powervr/`.
- **DRAM margin (important):** validate on a stable DDR configuration (e.g. native
  open-source `k1-ddr` at 2400 MT/s on 8 GB and 16 GB boards, or a conservative clock).
  Marginal DRAM causes single-bit flips in Client CCB command reads (for example, bit 2
  turning `GEOM` `0x2ABC80C9` into `CDM` `0x2ABC80CD`) that look like firmware bugs.
- **Hardware safety rules:**
  - **Never** read GPU MMIO registers while the GPU power island is off (check the SoC
    APMU power-domain status bit first, and hold `power/control = on` during any direct
    register sampling, then restore `auto`).
  - **Never** kill a running 3D/compute workload mid-job with `SIGKILL`/`timeout` unless
    explicitly testing HWR — abrupt teardown can leave a page fault/HWR behind.
  - On this SoC, GPU bus addresses `>= 0x80000000` map to `bus_addr + 0x80000000` in the
    CPU physical address space; direct CPU reads of `0x80000000..0xFFFFFFFF` lock up the
    SoC.

---

## 2. Pre-flight check on the `.fw` container (before booting on the board)

Before copying a newly built `.fw` file onto the board, verify its container layout
([memory-model](memory-model.md), `pvr_fw_info.h`):

1. Total file size is a multiple of `4096` bytes (`FW_BLOCK_SIZE`).
2. `pvr_fw_info_header` at `file_size - 4096` has:
   - `info_version == 3` (`PVR_FW_INFO_VERSION`);
   - `bvnc == 0x0024001D003400B6` (`36.29.52.182`);
   - `PVR_FW_FLAGS_OPEN_SOURCE` set in `flags`;
   - `fw_version_major >= 1`, `layout_entry_num <= 8`, valid `device_info_size`.
3. Layout entries (`MIPS_CODE`, `MIPS_EXCEPTIONS_CODE`, `MIPS_BOOT_CODE`,
   `MIPS_PRIVATE_DATA`, `MIPS_BOOT_DATA`, `MIPS_STACK`) have **no overlapping**
   `alloc_offset` ranges within code or data allocations.
4. Every ELF `PT_LOAD` segment fits inside its matching layout entry (`base_addr` +
   `max_size`) and ELF flags indicate microMIPS o32 (`0x72001001`).

---

## 3. Three-stage acceptance oracle

**Do not accept `rc == 0` alone.** A silent discard or timeout recovery can return `0`
without having executed real GPU work. Every stage below must verify **both** the
application output and the kernel/firmware trace (`/sys/kernel/debug/dri/*/pvr_fw/trace_0`
and `dmesg`).

### Stage A — Cold boot

Load the driver (`sudo dmesg -C && sudo modprobe powervr`) and verify in `dmesg`:

- `FW version v1.1` (matching your image version) and `Initialized powervr 1.0.0`.
- **Zero** occurrences of: `HW poll .* failed`, `BIF.*fault`, `MMU.*fault`, `FW fault`,
  `ASSERT`, `FWCCB.*112` / `type=112`, `device lost`, `error -22`, `Lockup`.

### Stage B — Fast functional sanity (all four DMs + SPM/OOM)

Run five short Vulkan workloads covering geometry, fragment, compute, 3D transfer, and
out-of-memory partial render:

| Workload type | Subsystems exercised | Required trace proof (per run) |
|---|---|---|
| Basic 3D renderpass | Geometry (TA) + Fragment (3D) | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| Geometry-heavy renderpass | Geometry-heavy TA + 3D | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| Compute dispatch | Compute (CDM, DM 4) | `rc == 0`, buffer check, `Kick Compute >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| 3D transfer / MSAA resolve | 3D Transfer (`TQ_3D`) | `rc == 0`, pixel/output check, `Kick 3D TQ >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| Parameter-Buffer OOM render | Multi-million-triangle PB OOM, freelist grow, multiple Partial Renders (`FRAG_PR`) | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |

> Note: ignore benign trace lines containing `HWRNeeded=0` or `HWRTData`; treat
> `HWRNeeded=1`, `Lockup`, `BIF.*fault`, `MMU.*fault`, `FW fault`, `Discarded cmd`,
> `FWCCB.*112`, `device lost`, and `error -22` as hard failures.

### Stage C — Pixel-checked Vulkan CTS (`deqp-vk`, 6 safe suites)

Run with `--deqp-log-images=disable --deqp-log-shader-sources=disable` and assert `rc == 0`,
`Failed: 0/<N>`, `Warnings: 0/<N>`, and `0` kernel faults in `dmesg`:

| Suite | Reference baseline on K1 mainline (**O**) |
|---|---|
| `dEQP-VK.api.smoke.*` | `Passed: 8/8`, `Failed: 0/8` |
| `dEQP-VK.draw.renderpass.simple_draw.*` | `Passed: 4/4`, `Failed: 0/4` |
| `dEQP-VK.compute.pipeline.basic.*` | `Passed: 73/82`, `Not supported: 9/82`, `Failed: 0/82` |
| `dEQP-VK.api.copy_and_blit.core.image_to_image.simple_tests.*` | `Passed: 23/23`, `Failed: 0/23` |
| `dEQP-VK.synchronization.basic.*` | `Passed: 24/29`, `Not supported: 5/29`, `Failed: 0/29` |
| `dEQP-VK.api.buffer.*` | `Passed: 1030/3076`, `Not supported: 2046/3076`, `Failed: 0/3076` |

> **Operational warning:** avoid running `dEQP-VK.memory.allocation.basic.*` unguarded on
> unified-memory K1 boards — it attempts to pin gigabytes of GPU buffers and can trigger a
> host Linux kernel OOM lockup unrelated to the GPU firmware. `dEQP-VK.api.buffer.*`
> exercises buffer creation and memory binding safely in under 10 seconds.

### Stage D — Runtime PM suspend and warm-boot resume

1. Ensure `/sys/bus/platform/devices/cac00000.gpu/power/control` is `auto`.
2. Wait for `/sys/bus/platform/devices/cac00000.gpu/power/runtime_status` to reach
   `suspended` (the GPU power island powers down after `FORCED_IDLE_REQ` + `OFF_REQ`).
3. Run a 3D transfer or renderpass workload from the `suspended` state, verifying that the
   firmware warm-boots cleanly (`rc == 0`, `Kick 3D TQ >= 1` or `Kick 3D >= 1`,
   `Discarded cmd == 0`, `0` faults, no `error -22` or `device lost`).
4. Unload the driver cleanly (`sudo modprobe -r powervr`) and restore the fallback firmware
   when finished.
