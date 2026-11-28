<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# Hardware recovery (HWR)

HWR (*hardware recovery*) detects a hung or faulting GPU and recovers it without bringing
down the firmware or the host.

> **Provenance.** Flags, enums, the HWR log layout and the cleanup contract: **P**
> (`pvr_rogue_fwif.h`, `pvr_fw.c`, `pvr_ccb.c`). Recovery flow and timing: **O** (trace) +
> **D**.

## Detection

- **MMU fault / page fault:** the MMU page-fault event in `ROGUE_CR_EVENT_STATUS` (or a
  diversion through the fault page, see [memory-model](memory-model.md)) indicates an
  invalid access by a job. The IRQ task sees the event.
- **Failed hardware poll:** the firmware polls GPU registers (e.g. waiting for a bit in
  `ROGUE_CR_EVENT_STATUS` after a kick). If a poll fails (unexpected value or timeout), the
  firmware records the poll address/mask/count in `rogue_fwif_sysdata.cr_poll_addr`,
  `cr_poll_mask`, `cr_poll_count` and triggers HWR (HWR type `ROGUE_HWRTYPE_POLLFAILURE`).
  Trace HWR id 13 `HW poll … failed (reg:… val:…)`.
- **Watchdogs:** the BG task checks the deadlines armed at each kick (see the per-DM
  execution state in [scheduler](scheduler.md)). A DM that does not finish in time is a
  lockup (or an overrun, `ROGUE_HWRTYPE_OVERRUN`).

## Recovery flow

```
detect a fault (MMU fault / poll / watchdog)
   └─ rogue_fwif_sysdata.hwr_state_flags: clear HARDWARE_OK, set RESET_IN_PROGRESS
      (+ GENERAL_LOCKUP / DM_STALLING / FW_FAULT as applicable)
   └─ hold scheduling (trace HWR id 8 "Hold scheduling due lockup …")
   └─ for each DM: "DM… ready for HWR" (HWR id 32)
   └─ identify the culprit: "Analysis DM…: Lockup FWCtx … Need to skip to next command" (HWR id 28)
   └─ "Begin hardware reset (HWR Counter=…)" (HWR id 61):
        reset the GPU core (ROGUE_CR_SOFT_RESET), flush/invalidate the SLC
   └─ "HWR: Reset HW" (MAIN id 27), "GPU Units init" (MAIN id 69)
   └─ "Finished hardware reset (HWR Counter=…)" (HWR id 62)
   └─ per-DM recovery:
        - discard the failed command ("Discarded cmd on DM… FWCtx=…", HWR id 14)
        - move the context to the next safe command ("Recovery DM…: FWCtx … skipped to
          command @ …", HWR id 24)
        - update hwr_recovery_flags[DM] and the HWR counters
        - "Recovery DM…: DM fully recovered" (HWR id 25)
   └─ ask the host for FREELISTS_RECONSTRUCTION if needed (see partial-render-spm)
   └─ hwr_state_flags = HARDWARE_OK (| DM_RUNNING_OK); reschedule
```

`hwr_state_flags` bits (public): `ROGUE_FWIF_HWR_HARDWARE_OK` bit 0,
`RESET_IN_PROGRESS` bit 1, `GENERAL_LOCKUP` bit 3, `DM_RUNNING_OK` bit 4, `DM_STALLING`
bit 5, `FW_FAULT` bit 6, `RESTART_REQUESTED` bit 7.

Observed: from lockup to recovery, about 0.43 ms. See
[observed-sequences](observed-sequences.md).

## Reset sequence (answers the clean team's Q11)

The complete ordered sequence run from the IRQ task once a lockup/fault is confirmed
(**D**/**O**):

1. **Stop the DMs** and hold scheduling (HWR state `RESET_IN_PROGRESS`).
2. **Wait for the SLC to go idle** — poll `ROGUE_CR_SLC_STATUS0` (bit 2
   `FLUSH_INVAL_PENDING` clear), after a flush/invalidate via `ROGUE_CR_SLC_CTRL_FLUSH_INVAL`.
3. **Disable partial renders** (`ROGUE_CR_PM_PARTIAL_RENDER_ENABLE = 0`).
4. **Pulse `ROGUE_CR_SOFT_RESET`** through the **XPU broadcast** bank, then release it:
   - assert: low word (`0x0100`) = `0x13FFFC1D`, high word (`0x0104`) = `0xFFEFFE00`;
   - release: both words = 0.

   These bits reset the GPU data-master/pipeline units **but not the MIPS core or the MTS
   (Garten) wrapper**, so the firmware keeps running. With `LAYOUT_MARS` present the MIPS
   core sits in its own (MARS) power/reset domain, consistent with these bits not touching
   it. The firmware does **not** use `ROGUE_CR_SOFT_RESET2` in recovery. (The larger
   "assert-and-hold" value `0x13FFFC1D`/`0xFFEFFE00` differs from the full
   power-on assert `0xF3FFFC1D`/`0xFFEFFFF8`, which is **not** used by HWR.)
5. **Re-initialise the GPU units** — re-run the whole GPU-units init (exec bases, SLC,
   unit defaults, event routing), i.e. section A of [boot-gpu-init](boot-gpu-init.md), and
   **rebuild the BIF page-catalogue slots** (`ROGUE_CR_BIF_CAT_BASE1..7`).
6. **Per-DM recovery:** skip/retire the guilty command, clear the DM lockup flags and its
   events (`ROGUE_CR_EVENT_CLEAR`), request freelist reconstruction if needed.
7. **Resume scheduling** — set `hwr_state_flags = HARDWARE_OK` and **kick the MTS**
   (`ROGUE_CR_MTS_SCHEDULE`).

So the clean team's draft order ("soft reset → SLC flush/invalidate → GPU units init →
re-enable MTS routing") is close but incomplete: the SLC wait and **disable partial
renders** come **before** the reset; the reset is a **pulse** (not assert-and-hold); after
it, **re-init the units and rebuild the BIF catalogue** and then do the **per-DM
skip/retire** before kicking the MTS. The MTS per-DM event routing is re-established as part
of the units re-init (section B of boot-gpu-init).

### Detailed bus quiesce and PM/TA state reset during HWR (answers Q29b item 4, D/O)

When a geometry/fragment lockup occurs, `SOFT_RESET` alone is not enough to leave PM/TA
usable for subsequent renders. The full per-core reset and PM/TA state teardown is:

1. **Quiesce BIF/MMU/SLC before `SOFT_RESET`:**
   - stall BIF/MMU requests (`CR[0x1340] |= 1`), poll `CR[0x1350] & 0xFF == 0` and
     `CR[0x1338] & 0xFFFF == 0`;
   - invalidate MMU caches (`0x1C`), issue a MIPS write barrier;
   - poll `CR[0x03C8] & 0x20 == 0x20` and `CR[0x3898] & 0xFF == 0xFF` (SLC idle).
2. **Two-stage `SOFT_RESET` assert and release:**
   - assert `BCAST[0x0100] = 0x13FFFC1D`, `BCAST[0x0104] = 0xFFEFFE00`, flush D-cache +
     write barrier;
   - OR in `0x10610000` (low) and `0x00080004` (high), then release both words to `0`.
3. **Post-reset unit & cache cleanup:**
   - re-run GPU-units init ([boot-gpu-init](boot-gpu-init.md) section A);
   - unstall BIF/MMU (`CR[0x1340] &= ~1`);
   - write `BCAST[0x1608] = 0x78001` and poll `0x1608` bit 0 == `0`;
   - flush/invalidate SLC (`BCAST[ROGUE_CR_SLC_CTRL_FLUSH_INVAL] = 1`, poll
     `ROGUE_CR_SLC_STATUS0` bit 2 == `0`);
   - rebuild the BIF page-catalogue table.
4. **Reset PM/SPM hardware and firmware tracking state:**
   - unpause PM allocation and deallocation: `BCAST[0x02A0] = CR[0x02A0] & ~3`;
   - disable partial renders: `BCAST[ROGUE_CR_PM_PARTIAL_RENDER_ENABLE] = 0`;
   - reset internal SPM state and SPM flags to `0`;
   - **clear all tracked active PM HWRTData slots (PM ctx 0 and 1) and all tracked loaded
     PM freelist slots to `0`** so the next geometry/fragment kick reloads them from scratch;
   - **request `ROGUE_FWIF_FWCCB_CMD_FREELISTS_RECONSTRUCTION`** from the host for every
     freelist that was loaded in any PM slot or attached to any active HWRTData. Because
     `SOFT_RESET` discards the PM's live page-allocation state, skipping freelist
     reconstruction leaves the freelists inconsistent and causes subsequent geometry kicks
     (including on HWRTData 0) to stall;
   - in DM 2 (GEOM) per-DM recovery, clear `hwrtdata.flags` bit 10 (`0x400`, mid-TA context
     store resume flag);
   - **do not clear `hwrtdata.geom_caches_need_zeroing` (`+0x114`)** during HWR. Leaving it
     `1` causes the **next** `FIRSTKICK` on that HWRTData to CPU-zero the TPC
     (`tail_ptrs_dev_addr`, `tpc_size`) and RTC (`rtc_dev_addr`, `0x300` bytes — which is
     what emits `Zero RTC for FWCtx`), followed by `CR[0x0258] = 1`, `0x0290`/`0x0198`, and
     `0x0C68 = 1` (see [kick-geometry-ta](kick-geometry-ta.md)).
   - HWR does **not** touch `0x0D20` directly; the next geometry kick reprograms `0x0D20`
     from `rta_ctl.active_render_targets`.

## HWR log (`rogue_fwif_hwrinfobuf`)

Each recovery is logged in a `rogue_hwrinfo` entry (see
[data-structures](data-structures.md)): fault data (union of BIF info — request status,
MMU status, page-catalogue address, which identifies the context —, MMU info, poll info,
MIPS TLB info), `cr_timer`/`os_timer`, `frame_num`, `pid`, `active_hwrt_data`,
`hwr_number`, `event_status` at the time, `hwr_recovery_flags`, **`hwr_type`**
(`enum rogue_hwrtype`, e.g. `ROGUE_HWRTYPE_BIF0FAULT = 3`, `ROGUE_HWRTYPE_MIPSTLBFAULT = 8`),
`dm`, `core_id`, and the `ROGUE_CR_TIMER` ticks of kick / reset start / reset finish /
freelist ready.

The buffer holds `ROGUE_FWIF_HWINFO_MAX` (16) entries: the first 8 are never overwritten,
the last 8 are a ring. Counters: `hwr_counter`, `write_index`,
`hwr_dm_locked_up_count[DM]`, `hwr_dm_overran_count[DM]`, `hwr_dm_recovered_count[DM]`, …

## Cleanup (KCCB `ROGUE_FWIF_KCCB_CMD_CLEANUP`)

The host asks for resources to be released with the KCCB `CLEANUP` command
(`ROGUE_FWIF_CLEANUP_FWCOMMONCONTEXT`, `_HWRTDATA`, `_FREELIST`, `_ZSBUFFER`). The firmware
may only clean an **idle** resource.

**Return contract (ABI, critical):** the return slot is a **bit mask**.

| Situation | Return slot |
|---|---|
| resource cleaned now **or already clean** | `ROGUE_FWIF_KCCB_RTN_SLOT_CMD_EXECUTED` (`0x1`) |
| resource still in use (pending submitted commands, DM running) | `CMD_EXECUTED \| ROGUE_FWIF_KCCB_RTN_SLOT_CLEANUP_BUSY` (`0x3`) — the host returns `-EBUSY` and retries |

The host waits for `CMD_EXECUTED` and then tests the `CLEANUP_BUSY` bit. Every path must
produce one of these results explicitly. Answering "busy" for a resource that is already
clean makes the host retry until it gives up, and the resource stays stuck.

## Context robustness (Vulkan device loss)

Contexts have `rogue_fwif_fwcommoncontext.robustness_address`. On a reset that affects a
context, the firmware can notify the host through FWCCB
`ROGUE_FWIF_FWCCB_CMD_CONTEXT_RESET_NOTIFICATION`, which reports device loss to the Vulkan
application.

## Interaction with power management

HWR and power management share state:

- during HWR **no** new work is scheduled (overlap rules, see [scheduler](scheduler.md));
- the power teardown walks the same BIF slot state that HWR resets (see
  [power-management](power-management.md)). Both paths must leave that state consistent;
- a power transition must not start while a recovery is in progress.

## Discarded geometry and subsequent 3D commands (answers Q42, D)

1. **Discard check on `hwrtdata.state == HWR (9)` before kicking 3D (or non-first TA):**
   at the very start of PM `HWRTData` loading (`load_hwrtdata(hwrtdata, new_state, ...)`),
   **before** touching any PM register:
   ```text
   if (new_state != HWRTDATA_STATE_KICKTAFIRST (2) &&
       hwrtdata.state == HWRTDATA_STATE_HWR (9))
       return DISCARD;
   ```
   - When a queued `FRAG` (`new_state == 4`) or non-first `GEOM` (`new_state == 3`) sees
     `hwrtdata.state == 9 (HWR)` (set by the host's freelist-reconstruction handler for every
     `HWRTData` using a reconstructed freelist — which is why holding DM 2/3 until
     `FREELISTS_RECONSTRUCTION_UPDATE` is required), the kick **must not touch the GPU**: it
     deactivates the BIF memory context, clears `fw_com_ctx_flags &= ~0xFF`, logs
     `HWR_DM_DISCARDED` (`Discarded cmd on DM3 (reason=1)`), and returns `DISCARD` so the
     dispatcher **retires the command and updates its UFOs/fences**.
   - On the next frame's `FIRSTKICK` (`new_state == 2`), `new_state == 2` bypasses the `== 9`
     check, transitions `hwrtdata.state` from `9 → 2`, and subsequent frames run cleanly
     without a second fault.
2. **Clearing `HWRTDATA_HAS_LAST_GEOM` when HWR retires a DM 2 command:**
   when HWR retires a running DM 2 geometry command during recovery, it clears bit 10 of
   `hwrtdata.flags` (`hwrtdata.flags &= ~0x400`); on DM 3 it clears bit 0 of
   `fw_com_ctx_flags` (`&= ~1`).
3. **Innocent / waiting DM 2 context during a DM 3 HWR:**
   at reset time, **every** DM with a running context (`running_ctx != NULL`) — except DM 3
   aborted mid-partial-render (`PR running`, which clears `running_ctx` without retiring the
   final 3D command) — is marked `NEEDS_SKIP` and **retired (discarded)** during recovery,
   **not** retried. Discarding the waiting DM 2 geometry is therefore normal; the follow-up 3D
   fault occurs only if the `hwrtdata.state == 9 (HWR)` discard check above is missing.
