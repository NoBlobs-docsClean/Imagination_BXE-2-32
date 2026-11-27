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

## 3. Acceptance oracle (Stages A–F)

**Do not accept `rc == 0` alone.** A silent discard or timeout recovery can return `0`
without having executed real GPU work. Every stage below must verify **both** the
application output and the kernel/firmware trace (`/sys/kernel/debug/dri/*/pvr_fw/trace_0`
and `dmesg`).

### Summary of required vs. optional acceptance criteria

| Stage | Scope | Mandatory for production firmware? |
|---|---|---|
| **A** | Cold boot (`dmesg` clean, `FW version v1.1`, zero faults/polls) | **Required** |
| **B** | Standalone Vulkan harness: multi-frame 3D (`HWRTData` alternation), heavy TA, compute, `TQ_3D` transfer, PB OOM + freelist grow + partial render (`FRAG_PR`), input-attachment subpasses, layered rendering (RTA) | **Required** |
| **C** | Pixel-checked Vulkan CTS (`deqp-vk`, 6 core suites + RTA / input-attachment checks) | **Required** |
| **D** | Runtime PM suspend (`suspended`) and warm-boot resume | **Required** |
| **E** | Controlled HWR injection (lockup / page fault) + freelist reconstruction + clean subsequent render on the same and a new device | **Required** |
| **F** | Extended 3D CTS (`indirect_draw`, `multisample`, `occlusion_query`) and priority preemption (`CSW` on 3D, TA, CDM) | **Recommended / Optional** |

### Stage A — Cold boot

Load the driver (`sudo dmesg -C && sudo modprobe powervr`) and verify in `dmesg`:

- `FW version v1.1` (matching your image version) and `Initialized powervr 1.0.0`.
- **Zero** occurrences of: `HW poll .* failed`, `BIF.*fault`, `MMU.*fault`, `FW fault`,
  `ASSERT`, `FWCCB.*112` / `type=112`, `device lost`, `error -22`, `Lockup`.

### Stage B — Fast functional sanity (all four DMs + SPM/OOM)

Run the standalone `vkfwtest` workloads (see Section 4 below) covering geometry, fragment,
compute, 3D transfer, and out-of-memory partial render:

| Workload | Subsystems exercised | Required trace proof (per run) |
|---|---|---|
| `pre-3d` | Geometry (TA) + Fragment (3D) | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| `pre-ta` | Geometry-heavy TA + 3D | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| `pre-cdm` | Compute (CDM, DM 4) | `rc == 0`, buffer check, `Kick Compute >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| `xfer` | 3D Transfer / MSAA resolve (`TQ_3D`) | `rc == 0`, pixel/output check, `Kick 3D TQ >= 1`, `Discarded cmd == 0`, `0` faults/HWR |
| `bigpb` | Multi-million-triangle PB OOM, freelist grow, multiple Partial Renders (`FRAG_PR`) | `rc == 0`, pixel/output check, `Kick TA >= 1`, `Kick 3D >= 1`, `Discarded cmd == 0`, `0` faults/HWR |

> Note: ignore benign trace lines containing `HWRNeeded=0` or `HWRTData`; treat
> `HWRNeeded=1`, `Lockup`, `BIF.*fault`, `MMU.*fault`, `FW fault`, `Discarded cmd`,
> `FWCCB.*112`, `device lost`, and `error -22` as hard failures.

### Stage C — Pixel-checked Vulkan CTS (`deqp-vk`)

Run with `--deqp-log-images=disable --deqp-log-shader-sources=disable` and assert `rc == 0`,
`Failed: 0/<N>`, `Warnings: 0/<N>`, and `0` kernel faults in `dmesg`:

| Suite | Reference baseline on K1 mainline (**O**) | Requirement |
|---|---|---|
| `dEQP-VK.api.smoke.*` | `Passed: 8/8`, `Failed: 0/8` | Required |
| `dEQP-VK.draw.renderpass.simple_draw.*` | `Passed: 4/4`, `Failed: 0/4` | Required |
| `dEQP-VK.compute.pipeline.basic.*` | `Passed: 73/82`, `Not supported: 9/82`, `Failed: 0/82` | Required |
| `dEQP-VK.api.copy_and_blit.core.image_to_image.simple_tests.*` | `Passed: 23/23`, `Failed: 0/23` | Required (catches TPU/NPOT/offset init bugs) |
| `dEQP-VK.synchronization.basic.*` | `Passed: 24/29`, `Not supported: 5/29`, `Failed: 0/29` | Required |
| `dEQP-VK.api.buffer.*` | `Passed: 1030/3076`, `Not supported: 2046/3076`, `Failed: 0/3076` | Required |
| `dEQP-VK.renderpass.suballocation.attachment.*input*` (e.g. `...d32_sfloat.input.clear.dont_care.draw`) | `Failed: 0` | Required (catches multi-subpass / second-HWRTData faults) |
| `dEQP-VK.api.image_clearing.core.clear_color_attachment.cube_layers.*` | `Failed: 0` | Required for v1.1 (`gs_rta_support` / layered rendering) |
| `dEQP-VK.draw.renderpass.indirect_draw.*`, `dEQP-VK.pipeline.monolithic.multisample.*`, `dEQP-VK.query_pool.occlusion_query.*` | `Failed: 0` | Recommended (Stage F) |

> **Operational warning:** avoid running `dEQP-VK.memory.allocation.basic.*` unguarded on
> unified-memory K1 boards — it attempts to pin gigabytes of GPU buffers and can trigger a
> host Linux kernel OOM lockup unrelated to the GPU firmware. `dEQP-VK.api.buffer.*`
> exercises buffer creation and memory binding safely in under 10 seconds.

### Stage D — Runtime PM suspend and warm-boot resume

1. Ensure `/sys/bus/platform/devices/cac00000.gpu/power/control` is `auto`.
2. Wait for `/sys/bus/platform/devices/cac00000.gpu/power/runtime_status` to reach
   `suspended` (the GPU power island powers down after `FORCED_IDLE_REQ` + `OFF_REQ`).
3. Run a transfer or 3D workload (`vkfwtest xfer`) from the `suspended` state, verifying
   that the firmware warm-boots cleanly (`rc == 0`, `Kick 3D TQ >= 1`,
   `Discarded cmd == 0`, `0` faults, no `error -22` or `device lost`).
4. Unload the driver cleanly (`sudo modprobe -r powervr`) and restore the fallback firmware
   when finished.

### Stage E — Controlled HWR and post-recovery sanity

Verify that a real GPU fault or lockup recovers cleanly **without** leaving PM/TA wedged:

1. Trigger a controlled lockup (e.g. an infinite-loop fragment/compute shader) or page fault
   in a disposable Vulkan device.
2. Verify in `dmesg` / trace:
   - HWR triggers, runs `SOFT_RESET` + GPU-units re-init + `FREELISTS_RECONSTRUCTION` (when
     a render was active), and reports `VK_ERROR_DEVICE_LOST` to the guilty device.
   - Zero firmware asserts or `FW fault`.
3. Immediately afterwards — **without reloading `powervr`** — run a multi-frame 3D render on
   a fresh Vulkan device and verify `rc == 0`, correct pixels, and zero subsequent lockups
   or faults (catches incomplete PM/TA teardown or missing freelist reconstruction).

### Stage F — Optional priority preemption (`CSW`) and stress

- Run concurrent low-priority (long) and high-priority (short) queues on the same DM
  (3D, TA, and CDM) using `VK_EXT_global_priority` (`LOW` vs `HIGH`) and confirm the
  high-priority job completes before the low-priority job finishes, with `CSW` store/resume
  activity in the trace and correct output from both jobs.
- Run a sustained multi-frame visual workload (e.g. `vkcube` or continuous offscreen loop for
  ≥ 100 frames) and verify stable frame completion and zero faults.

---

## 4. Implementing a standalone Vulkan firmware test harness - vkfwtest

To run Stages B, E, and F quickly without relying solely on `deqp-vk`, implement a small,
self-contained C/C++ Vulkan 1.1 CLI tool (single binary, headless — no window system needed,
linked only against `-lvulkan -lpthread`).

### Common scaffolding

1. **Headless Vulkan setup:**
   - Create a `VkInstance` and pick the `PowerVR` `VkPhysicalDevice`.
   - Create a `VkDevice` with a graphics + compute + transfer queue. For preemption tests,
     enable `VK_EXT_global_priority` and create two separate `VkDevice` instances (one with
     `VK_QUEUE_GLOBAL_PRIORITY_LOW_EXT`, one with `VK_QUEUE_GLOBAL_PRIORITY_HIGH_EXT`) so
     the kernel creates distinct firmware render/compute contexts with different priorities.
   - Use host-visible staging buffers (`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
     VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`) to read back rendered images or SSBOs and check
     exact pixel/word values on the CPU.
2. **Exit contract:**
   - Return `0` **only** if every submitted fence signals within timeout **and** CPU readback
     matches the expected colour/checksum. Return non-zero on `VK_ERROR_DEVICE_LOST`,
     timeout, or output mismatch.

### Recommended subcommands and how to build each one

| Subcommand | Firmware paths exercised | Minimal Vulkan implementation |
|---|---|---|
| `multiframe` (or `pre-3d`) | Multi-frame TA + 3D, **alternating `HWRTData 0/1`**, optional 3D preemption (`CSW`) | Allocate a 256×256 `R8G8B8A8_UNORM` colour image and a simple fullscreen-quad graphics pipeline (solid colour or uniform-driven colour). Submit **at least 6 sequential frames** on the **same `VkDevice`** (each frame: `vkCmdBeginRenderPass` → `vkCmdDraw(3 or 6, 1, 0, 0)` → `vkCmdEndRenderPass` → copy to staging → wait fence → verify pixels). Alternating frames force Mesa to ping-pong between the render target's two `HWRTData` instances. For optional **3D preemption**, run a long fragment shader (many ALU loop iterations per pixel) on a `LOW`-priority device in thread 1 while thread 2 submits short draws on a `HIGH`-priority device and checks latency + both outputs. |
| `pre-ta` | Heavy geometry (`TA`), multi-kick TA, optional TA preemption (`CSW`) | Render a dense grid of tiny triangles (e.g. 50k–200k triangles via `vkCmdDraw` with procedural `gl_VertexIndex` in the vertex shader — no vertex buffer needed) into a 512×512 target across multiple frames, checking output pixels. For optional **TA preemption**, run a heavy geometry draw on a `LOW`-priority device concurrently with a light draw on a `HIGH`-priority device. |
| `pre-cdm` | Compute (`CDM`, DM 4), optional CDM preemption (`CSW`) | Create a compute pipeline that writes a deterministic pattern (`buf[id] = id ^ seed`) to an SSBO (`vkCmdDispatch(64, 1, 1)`), wait on the fence, and verify every element on the CPU. For optional **CDM preemption**, run a long compute loop on a `LOW`-priority device while a `HIGH`-priority device dispatches a short compute job. |
| `xfer` | 3D Transfer (`TQ_3D`) + TPU/PBE init | Create two `R8G8B8A8_UNORM` images, including a **non-power-of-two (NPOT)** size (e.g. 137×91) and a power-of-two size (256×256). Fill the source via staging buffer with a byte-unique pattern (`(x + y*width) ^ const`), execute `vkCmdBlitImage` / `vkCmdCopyImage` with sub-region offsets and a flipped/mirrored region, read back the destination, and verify every byte. |
| `bigpb` | Parameter-Buffer **OOM**, `FREELIST_GROW_UPDATE`, multiple **Partial Renders (`FRAG_PR`)**, TA resume | Render into a **1024×1024** `R8G8B8A8_UNORM` target with a **depth attachment** (`D32_SFLOAT`, `LOAD_OP_CLEAR`, `STORE_OP_STORE` so ZLS is backed for partial renders). Draw **2–5 million small triangles** covering the viewport in a single render pass (using `gl_VertexIndex` in the vertex shader to tile tiny triangles across all 16×16 tiles, forcing the PM to exhaust the initial parameter buffer, request freelist grows, and trigger multiple partial renders). Verify the final colour buffer matches the expected colour with zero black/missing tiles. |
| `input-attach` | Multi-subpass render pass (two `HWRTData` / IPF read of subpass 0 output) | Single `VkRenderPass` with **2 subpasses** and a colour + depth (`D32_SFLOAT`) attachment: subpass 0 clears and draws a quad; subpass 1 reads subpass 0's attachment via `subpassLoad()` as a `VK_DESCRIPTOR_TYPE_INPUT_ATTACHMENT` and writes a derived colour to a second colour attachment. Read back and verify every pixel. |
| `rta` | Firmware v1.1 layered rendering (`gs_rta_support`, `rta_ctl`) | Create a 2D array image (`arrayLayers = 6`, e.g. 64×64) and a framebuffer with `layers = 6`. Use a geometry or vertex shader (`VK_EXT_shader_viewport_index_layer` / `gl_Layer`) to render a distinct solid colour into each layer `0..5` in one render pass. Read back all 6 layers and verify each layer's colour. |
| `hwr-lockup` | Controlled HWR + `FREELISTS_RECONSTRUCTION` + post-HWR recovery | On device 1, submit a compute or fragment shader with an infinite loop (guarded by a uniform buffer value `1` so the compiler cannot optimize it away) and wait for `vkWaitForFences` to return `VK_ERROR_DEVICE_LOST`. Destroy device 1, immediately create device 2 (without reloading `powervr`), run `multiframe`, and require `rc == 0` with valid pixels. |

### Minimal shader tip

Embed precompiled SPIR-V arrays directly as `static const uint32_t spv_...[]` (compiled
offline with `glslangValidator -V -x`) so the harness has **zero** runtime shader-compiler
dependencies on the target board.
