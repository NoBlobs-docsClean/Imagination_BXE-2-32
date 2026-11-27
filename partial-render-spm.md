<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

# Partial render and out-of-memory handling (SPM)

Geometry consumes **Parameter Buffer (PB)** pages from a **freelist**
(`rogue_fwif_freelist`). When the freelist runs out in the middle of geometry, the PM
hardware signals an out-of-memory event. The firmware's out-of-memory / partial-render
handling (trace group `ROGUE_FWIF_LOG_TYPE_GROUP_SPM`, here simply "SPM") decides what to do
so that the frame is not lost.

> **Provenance.** Protocol and command names: **P** (`pvr_rogue_fwif.h`, `pvr_free_list.c`,
> `pvr_ccb.c`). States: **P** (public trace strings) + **O** (seen in the trace). Ordering
> rules: **D**.

## Decision on OOM

On an out-of-memory event during geometry (trace SPM id 7 `OOM TA_cmd=…, U-FL …, N-FL …`),
the firmware chooses:

1. **Use the ready pages** of the freelist (`rogue_fwif_freelist.ready_pages`, pages
   reserved for OOM), if any: the geometry resumes (trace SPM id 44 `OOM: Resumed TA with
   ready pages, …`).
2. **Grow the freelist** (preferred when possible):
   - send FWCCB `ROGUE_FWIF_FWCCB_CMD_FREELIST_GROW` to the host, mark
     `rogue_fwif_freelist.grow_pending` and enter the *wait for grow* state (trace SPM id 34,
     id 41 `Wait for pending grow on Freelist …`);
   - the host allocates more pages and answers with KCCB
     `ROGUE_FWIF_KCCB_CMD_FREELIST_GROW_UPDATE` (trace SPM id 54 `Received grow update, …`);
   - the geometry resumes.
3. **Partial render** (when growing is not possible):
   - run the `ROGUE_FWIF_CCB_CMD_TYPE_FRAG_PR` command of the context's cCCB: a partial
     render of what has already been tiled, which frees PB pages;
   - **ZS/MSAA** buffers that are needed are requested on demand through FWCCB
     `ROGUE_FWIF_FWCCB_CMD_ZSBUFFER_BACKING` (the host confirms with KCCB
     `ROGUE_FWIF_KCCB_CMD_ZSBUFFER_BACKING_UPDATE`);
   - after the partial render, geometry **restarts** where it stopped.

## Observed sequences (answers Q23, O)

Black-box observation of the open v1.1 image on the board running `vkfwtest bigpb`: one
1024×1024 render target, millions of triangles per frame, so geometry runs out of PB pages
again and again. The quoted lines are public trace strings.

**Freelists involved.** The trace names both freelists on every OOM: `U-FL` (the
**global** freelist) and `N-FL` (the HWRTData's **local** freelist). In this run the local
freelist had `max = total = 32` (grow size 0) and was never extended. **Every extension
and grow went to the global freelist.** When the geometry kick loads them, the trace
prints `UFL-TA-Base: <global base> (SP = …)` and `FL-TA-Base: <local base> (SP = …)`.

**1. OOM with ready pages: no host round trip.**
```
"Out of memory! Context …, HWRTData …"
"PM pause TA ALLOC: PM_PAGE_MANAGEOP set to 0x1"
"PM unpause TA ALLOC: PM_PAGE_MANAGEOP set to 0x0"
"Freelist update completed [global]: old total pages 0x1c0, new total pages 0x200,
 new DevVirtAddr 0x…48800"                      (was 0x…48900)
"OOM: Resumed TA with ready pages, FL addr: 0x…48800, current pages: 512, SP : 511"
```
- The firmware adds the freelist's `ready_pages` (64 here) to its total.
- The freelist's base address moves **down by 4 bytes per added page**: the page stack grows
  toward lower addresses.
- The geometry then **resumes**; no new `Kick TA` appears.
- Allocation is paused through `PM_PAGE_MANAGEOP` (`CR[0x02A0]`) while the PM state is
  changed.

**2. OOM without ready pages: grow.**
```
"Out of memory! …"
"OOM TA_cmd=…, U-FL <global>, N-FL <local>"
"OOM TA/3D PR Check: [<ufo>] is 0x0 requires 0x0"
"Wait for pending grow on Freelist <global>"
"SPM State = wait for grow"                     (FWCCB FREELIST_GROW to the host)
"Received grow update, FL addr: …, new pages: 768, ready pages: 96"   (KCCB FREELIST_GROW_UPDATE)
"PM pause TA ALLOC …" / "PM unpause TA ALLOC …"
"Freelist update completed [global]: old total 0x200, new total 0x2a0, new DevVirtAddr …"
"SPM State = PR avoided"
"Resume TA without partial render"
"SPM State = none"
```
After a grow update with `new pages = 768` and `ready pages = 96`, the new current total is
`768 − 96 = 672` (`0x2a0`). The ready pages are **held back** and handed out at the next OOM
(path 1: `0x2a0 → 0x300`).

**3. OOM that cannot grow: partial render.** This happens once the global freelist has
reached its maximum (65536 pages in this run).
```
"Out of memory! …"
"OOM TA_cmd=…, U-FL …, N-FL …"
"OOM TA/3D PR Check: [<ufo>] is 0x3 requires 0x3"     (geometry-fragment fence passes)
"No deferred partial render FW (Type=0) Buffer provided"
"No deferred partial render FW (Type=1) Buffer provided"
"SPM State = PR blocked"
"SPM State = wait for HW"                       (wait until the fragment DM is free)
fragment-side load: "Loading stack-pointers for 1 (0:MidTA,1:3D) on context 1, MLIST = …,
                     ALIST = …", "3D RTData … ready on HW context 1"
"UFL-3D-Base: <global> (SP = 9, …)", "ALIST1 SP = …, MLIST1 SP = … (VCE …, TE …, ALIST …)"
"Kick 3D: … Partial render:1 …"
"SPM State = PR running"
"3D finished, …"
"UFL-3D-Base: <global> (SP = 65527, …)"         (pages returned to the global freelist)
"Partial Render finished"
"SPM State = PR executed"
"Restarting TA after partial render, HWRTData0State=0x8, HWRTData1State=0x8"
"Restart TA after partial render"
"SPM State = none"
```
- The partial render is the context's `FRAG_PR` command run on the fragment DM, with PM
  context 1 loaded from the geometry's current ALIST/MLIST stack pointers and page
  catalogues.
- When it finishes, the pages come back (SP 9 → 65527), and the geometry is **restarted
  without a new `Kick TA`**.
- The `PR Check` line is the test of `partial_render_geom_frag_fence` (see
  [client-ccb](client-ccb.md)).
- "No deferred partial render FW Buffer provided" appears when the geometry command's
  `pr_buffer_fw_addr[]` entries are 0.

**The "4PB/4PT" counters.** The `UFL-…-Base`/`FL-…-Base` lines print `SP`, `4PB` and `4PT`
per freelist, for example `SP = 9, 4PB = 65402, 4PT = 124` near exhaustion. These are the
stack pointer and the pages handed out for parameter-buffer data and for page tables. They
correspond to the `Alloc Page Count` and `Alloc MMU Page Count` of the `Load`/`Store
Freelist` lines (local freelist registers in [kick-geometry-ta](kick-geometry-ta.md), Q16b).

**Not exercised in this run (`PENDING`):** on-demand ZS backing
(`ZSBUFFER_BACKING`/`_UPDATE`; no `pr_buffer` was provided), and freelist reconstruction
after HWR (no HWR occurred). The global freelist's registers (base and page counters, one
bank per PM context: `0x20B8` geometry, `0x2078` fragment) are given in
[kick-geometry-ta](kick-geometry-ta.md) (Q16e–Q16g), including the live stack pointers
(global `0x20C8`/`0x2088`, local `0x0348`/`0x0350`).

## Resuming geometry after OOM (answers Q28, D/O)

1. **Which freelist is extended on OOM:** the firmware does **not** read a hardware
   register to identify which freelist ran out. It always selects the HWRTData's **global**
   freelist: `freelists_fw_addr[1]` for the primary geometry pipe (or `freelists_fw_addr[2]`
   when `hwrtdata.flags & 0x100` is set for a secondary geometry pipe).
2. **Resuming with `ready_pages` (`OOM: Resumed TA with ready pages`):**
   after pausing (`CR[0x02A0] = 1`, poll `CR[0x02A8] & 1 == 1`), updating the freelist base
   and stack pointer, and unpausing (`CR[0x02A0] = 0`, poll `CR[0x02A8] & 1 == 0`), write:
   ```text
   CR[0x0328] = 1          // primary core bank — resumes the PM / TA after freelist update
   ```
   This single write is what makes the stalled geometry continue (neither re-clearing
   `PM_OUT_OF_MEMORY` nor re-writing `CR[0x0400] = 1` resumes it).
3. **Pausing geometry when no `ready_pages` are available (entering `wait for grow` or
   `PR blocked`):**
   ```text
   CR[0x0320] = 1                          // request TA stop on OOM
   poll CR[0x0130] (EVENT_STATUS) bit 6 = 1  // TA_TERMINATE
   CR[0x0138] = 0x40                       // clear EVENT_STATUS bit 6
   CR[0x0CF0] = 1                          // pause TE/TA state
   poll CR[0x0CF0] & 1 == 0
   ```
4. **Resuming geometry after `FREELIST_GROW_UPDATE` (`SPM State = PR avoided` →
   `Resume TA without partial render`):**
   set `hwrtdata.state = 1`, then write both:
   ```text
   CR[0x0328] = 1          // resume PM
   CR[0x0CF8] = 1          // resume TE/TA paused by 0x0CF0
   ```
5. **Restarting geometry after a partial render (`SPM State = PR executed` →
   `Restart TA after partial render`, answers Q30 items 1–3, D/O):**
   - **At `3D finished` of the `FRAG_PR`:**
     1. wait for `PM_3D_MEM_FREE` (`EVENT_STATUS` bit 8) and clear it, run
        `hwrtdata` 3D-done (`hwrtdata.state = 8` + full 3-step PM-MMU invalidation), clear
        `PIXELBE_END_RENDER` (and `ZLS_FINISHED` if set), and run the DM 3 cache flush;
     2. write **`CR[0x0338] = 0`** (`ROGUE_CR_PM_PARTIAL_RENDER_ENABLE`), set SPM state to
        `PR executed`, and set **`hwrtdata.flags |= 0x8`** (`+0xA8`, marks that at least one
        partial render has executed on this HWRTData).
   - **In the TA-restart handler (primary core bank, exact order):**
     1. **Save 3D PM context catalogue bases and stack pointers** (when `active_hwrtdata[1]
        == this_hwrtdata` and `hwrtdata.state == 8`): read PM context 1's live registers
        and write them back into `rogue_fwif_hwrtdata`:
        - `hwrtdata+0x08` (`vce_cat_base[0]`) ← 64-bit `CR[0x1268/0x126C]` (`PM_CAT_BASE_VCE1`)
        - `hwrtdata+0x48` (`te_cat_base[0]`) ← 64-bit `CR[0x1270/0x1274]` (`PM_CAT_BASE_TE1`)
        - `hwrtdata+0x88` (`alist_cat_base`) ← 64-bit `CR[0x1280/0x1284]` (`PM_CAT_BASE_ALIST1`)
        - `hwrtdata+0x98/0x9C` (`pm_alist_stack_pointer`) ← 64-bit `CR[0x0280/0x0284]`
        - `hwrtdata+0xA0` (`pm_mlist_stack_pointer`) ← 32-bit `CR[0x02D0]`
        Then clear `active_hwrtdata[1] = 0`.
     2. Set `hwrtdata.state = 1`.
     3. Select PM context 0 in `0x03D0`/`0x03D4`:
        `CR[0x03D0] &= ~0x00010000`, `CR[0x03D4] &= ~0x00000100`.
     4. Write **`CR[0x0290] = 1`**, then **`CR[0x0198] = 1`** (no poll between them).
     5. Run the **full 3-step PM-MMU invalidation** (not `0x1348` alone):
        - `BCAST[ROGUE_CR_SLC_CTRL_FLUSH_INVAL] = 0x10`, poll `ROGUE_CR_SLC_STATUS0`
          (`0x3820`) bit 2 == `0`;
        - `BCAST[ROGUE_CR_BIF_CTRL_INVAL] = 4`, poll `CR[0x12A0]` bit 2 == `0`;
        - if GPU power domain is on: `BCAST[0x1348] = 1`, poll `CR[0x1348]` bit 0 == `0`.
     6. Write **`CR[0x01B8] = 1`** and poll **`CR[0x01B8] & 1 == 0`**. If this poll times
        out, the firmware records `ROGUE_HWRTYPE_POLLFAILURE` (`cr_poll_addr = 0x01B8`),
        triggers **HWR**, and returns immediately without writing `0x0328`/`0x0CA8`.
     7. Write **`CR[0x0328] = 1`**, then **`CR[0x0CA8] = 1`** to restart the TA.
     8. Clear SPM state (`none`) and re-arm the DM 2 watchdog.

### Multi-PR kick state and ZLS / background-object rules (answers Q30 items 2 & 4, D/O)

1. **Mandatory PM partial-render register on `FRAG_PR` (`CR[0x01C0]`):**
   every fragment kick must write **`CR[0x01C0] = 1`** when running a partial render
   (`FRAG_PR`) and **`CR[0x01C0] = 0`** on a normal 3D render (primary core bank, alongside
   `CR[0x0338] = 1` on `FRAG_PR`, and `CR[0x0298] = 0`, `CR[0x01C8] = 0` when not resuming
   a 3D context store). Omitting `CR[0x01C0] = 1` during `FRAG_PR` leaves PM state
   inconsistent across multiple partial renders on the same HWRTData and causes `0x01B8`
   (or the SLC/PM-MMU invalidation) to stall on the second PR restart.
2. **PM context 1 state on the 2nd (and subsequent) `FRAG_PR`:**
   PM context 1 always takes the **geometry's current catalogues and ALIST/MLIST stack
   pointers again** (it does **not** continue from the previous PR's state). Because the
   geometry HWRTData remains in `active_hwrtdata[0]` while TA is paused in OOM, loading the
   HWRTData into PM context 1 first saves PM context 0's live catalogues/stack pointers
   (`0x1248`/`0x1250`/`0x1260`, `0x0270/0x0274`, `0x02C0`) into `rogue_fwif_hwrtdata`
   (without clearing `active_hwrtdata[0]` while SPM is active) and then loads them fresh
   into PM context 1 (`active_hwrtdata[1]` was cleared at the end of the previous PR).
3. **ZLS and background-object programming for `FRAG_PR` and final 3D after a PR:**
   Let `has_pr = (hwrtdata.flags & 0x8) != 0` (0 on the 1st `FRAG_PR`; 1 on the 2nd+
   `FRAG_PR` and on the final normal 3D render after any PR).
   - **ZLS base addresses:** always program `cmd_3d.isp_zload_store_base` to both
     `BCAST[0x0F50/0x0F54]` and `BCAST[0x0F58/0x0F5C]`, and `cmd_3d.isp_stencil_load_store_base`
     to both `BCAST[0x0F60/0x0F64]` and `BCAST[0x0F68/0x0F6C]`. On-demand
     `ZSBUFFER_BACKING` is only requested when `cmd_geom.pr_buffer_fw_addr[]` is non-zero;
     when the host already backed the depth/stencil buffer (as in `vkfwtest bigpb`),
     `pr_buffer_fw_addr[]` is 0 and the command's own ZLS bases are used directly.
   - **Forced Z/S store on every `FRAG_PR`:**
     - if `cmd_3d.flags & 0x80` (`DEPTHBUFFER`): `isp_zlsctl_lo |= 0x80004` (Z-store + ZLS enable);
     - if `cmd_3d.flags & 0x100` (`STENCILBUFFER`): `isp_zlsctl_lo |= 0x40004` (S-store + ZLS enable);
     - when either bit is set, OR `0x20` into `BCAST[0x06D0]` (`0x3F` instead of `0x1F`,
       unless `cmd_3d.flags & 0x8000` forces `0x7F`).
   - **Forced Z/S load and `pds_pr_bgnd` whenever `has_pr != 0` (2nd+ `FRAG_PR` and final 3D):**
     - if `cmd_3d.flags & 0x80` (`DEPTHBUFFER`): `isp_zlsctl_lo |= 0x8002` (Z-load + ZLS enable);
     - if `cmd_3d.flags & 0x100` (`STENCILBUFFER`): `isp_zlsctl_lo |= 0x4002` (S-load + ZLS enable);
     - load `BCAST[0x06A0..0x06BC]` from **`pds_pr_bgnd0..3`** (`cmd_3d + 0x1C8..0x1DF`)
       instead of `pds_bgnd0..3`, and set `BCAST[0x0F88] |= 0x200`;
     - on the **1st** `FRAG_PR` (`has_pr == 0`), keep the normal `pds_bgnd0..3` (`cmd_3d +
       0x198..0x1AF`) and do **not** force Z/S load (only store).
     - `ROGUE_CR_ISP_CTL`: on the 1st `FRAG_PR` (`has_pr == 0`), if `pds_bgnd !=
       pds_pr_bgnd` or `cmd_3d.flags & 0x80000`, set bit 17 (`|= 0x20000`); when `has_pr !=
       0`, clear bit 17 unless `cmd_3d.flags & 0x100000` (or `cmd_3d.flags & 0x80000` on
       the final normal 3D).

## States and flags

The public trace strings name the states the SPM goes through (trace group SPM):

| Trace id | State |
|---|---|
| 32 | none |
| 33 | PR blocked |
| 34 | wait for grow |
| 35 | wait for HW |
| 36 | PR running |
| 38 | PR executed |

- The SPM state is kept per geometry context / freelist. The scheduler respects it: it does
  not schedule the final fragment while a partial render is pending (interlocks in the
  inter-DM overlap rules, see [scheduler](scheduler.md)).
- In the *wait for HW* state, with no 3D context running, the partial render is started
  and DM 3 is marked pending.
- The UFO checks that decide whether a PR may run are traced as SPM id 66 `OOM TA/3D PR
  Check: …`.
- The OS runtime flags (`rogue_fwif_os_runtime_flags`) include `fl_grow_pending` (bit 4):
  a freelist grow is in progress.

## Freelist reconstruction after HWR

After a hardware recovery that corrupted freelists, the firmware sends FWCCB
`ROGUE_FWIF_FWCCB_CMD_FREELISTS_RECONSTRUCTION` and the host answers with KCCB
`ROGUE_FWIF_KCCB_CMD_FREELISTS_RECONSTRUCTION_UPDATE` (failures flagged with
`ROGUE_FWIF_FREELISTS_RECONSTRUCTION_FAILED_FLAG`). See
[hardware-recovery](hardware-recovery.md).

## Fidelity notes (observed behaviour)

- The FREELIST_GROW protocol is a **strict request ↔ confirmation pair**. The firmware
  keeps the context blocked until the confirmation arrives.
- **PM pause handshake during freelist updates (O/D):** before changing the freelist base
  or stack pointer in hardware, set `CR[0x02A0] = 1` (pause TA allocation), poll
  `CR[0x02A8]` until bit 0 reads `1`, update the freelist state, then write `CR[0x02A0] = 0`
  to unpause.
- **Non-mutating fence evaluation when arming a partial render (D/O):** while in
  `PR blocked`, check whether the fragment context's `FENCE_PR` + `FRAG_PR` can run
  **without mutating the live `dep_offset` in shared memory** until all `FENCE_PR` UFOs
  satisfy the SPM failing-UFO check and any required ZS/MSAA `PRBUFFER` is backed. Only
  once those conditions hold should the firmware record the PR command offset, transition
  to `wait for HW`, and schedule DM 3.
- **Invalidate the active 3D HWRTData slot after each partial render (D/O):** the firmware
  tracks which `rogue_fwif_hwrtdata` is currently loaded in each PM hardware context
  (geometry / 3D / secondary geometry). When restarting geometry after a partial render
  completes (`PR executed` → `none`), **clear/invalidate the 3D PM slot** if it still
  references that HWRTData. Otherwise, a subsequent partial render on the same HWRTData
  (common in multi-PR frames such as `vkfwtest bigpb`) sees a false cache hit on the 3D PM
  slot, skips reloading the updated `ALIST1`/`MLIST1` stack pointers, and immediately
  triggers a `BIF0` page fault.
- Freelist cleanup requests must return the **correct return-slot bits**:
  `ROGUE_FWIF_KCCB_RTN_SLOT_CMD_EXECUTED` alone when the freelist was cleaned, and
  `CMD_EXECUTED | ROGUE_FWIF_KCCB_RTN_SLOT_CLEANUP_BUSY` when it is still in use. Reporting
  "busy" wrongly makes the host retry and give up, leaving the freelist stuck.
