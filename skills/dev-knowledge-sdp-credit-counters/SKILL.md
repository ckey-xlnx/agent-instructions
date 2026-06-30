---
name: dev-knowledge-sdp-credit-counters
description: >
  Reference for the debug tools that observe SDP credit grant/consume behaviour on
  MI450: the SDPSW (SDP switch) perf counters, scan dumps, and host IFoE telemetry.
  Use when reading or producing dumps from the sdpsw_cg_disable_with_perf_cnts_v2.py
  debug script, picking SDPSW perf event numbers, or tabulating granted-vs-used
  credit per station/interface along the DXS->SDPSW->IFoE path. Covers the
  SION==SDPSW / CAKE==IFoE naming, the per-edge event-number map (DX_SION_*,
  SION_DX_*, SION_CAKE_*, CAKE_SION_*, CHAININ/CHAINOUT), which event pairs give
  "used" (Vld) vs "granted" (CreditVld) for Req and MstrData, how to read held
  credit = grant - used, the expected IFoE initial credit (14 req / 18 data per
  connect), the MID0=CPU / MID1=IFoE station split, and the two other data sources
  (scan dumps, tech-support telemetry). Trigger phrases: "SDP credit counter",
  "SDPSW perf counter", "ReqCreditVld", "MstrDataCreditVld", "DX_SION", "SION_CAKE",
  "SION==SDPSW", "CAKE==IFoE", "sdpsw_cg_disable_with_perf_cnts", "credit granted vs
  used", "IFoE credit debug", "sdpsw_cnt0", "perf event number".
---

# SDP Credit Counter Debug (MI450)

The debug tools for capturing and interpreting SDP **credit** flow on MI450 —
granted (credit issued by the consumer) vs used (messages sent by the producer).
For the underlying credit model (who
grants, who holds, initial credit, remember/forget across disconnect) see
`dev-knowledge-sdp-interface`. For the GPU-internal SDP topology (DXS/DXM, SDP
switch, DF) see `dev-knowledge-mi450-gpu`; for the IFoE blocks see
`dev-knowledge-ifoe-core` / `dev-knowledge-ifoe-subsystem`.

## Three data sources

1. **SDPSW perf counters** — the `sdpsw_cg_disable_with_perf_cnts_v2.py` script
   (below). Live register reads of the SDP-switch event counters. Primary tool.
2. **Scan dumps** — post-mortem scan of internal state (same block, captured via
   scan rather than register reads).
3. **Host IFoE telemetry** — IFoE debug counters exposed to the host as telemetry
   and captured in "tech support" files.

This skill focuses on (1); (2) and (3) are alternative views of related state.

## The script

`sdpsw_cg_disable_with_perf_cnts_v2.py` (PAPI2 / toollib based). Source of truth
for event-number -> name mapping is the `SDPSW_COUNTERS` dict inside the script.

- Authoritative latest copy:
  `http://dcgpuval-storage.amd.com/users/swarrick/MI450_Debug_Scripts/Latest/sdpsw_cg_disable_with_perf_cnts_v2.py`
- Author: swarrick. The script also handles L1BIU (IOMMU) perf counters via a
  separate `--*_biu_*` set of flags and its own `BIU_PERF0/1_COUNTERS` dicts.

### Counter slots and the read pipeline

The SDPSW block has **4 event-select slots** (cnt0..cnt3). You program each slot
with an **event number**, run traffic, then read the 4 accumulated counts. Each
read targets every SDP_SW station instance.

Typical capture sequence (run as `sudo python3 ... -a toollib`):

```
--rst_sdpsw_perf True                                              # clear
--en_sdpsw_perf True --sdpsw_cnt0 N0 --sdpsw_cnt1 N1 --sdpsw_cnt2 N2 --sdpsw_cnt3 N3   # program + start
--rd_sdpsw_perf  True --sdpsw_cnt0 N0 --sdpsw_cnt1 N1 --sdpsw_cnt2 N2 --sdpsw_cnt3 N3   # read (repeat around each op)
```

> **Gotcha — the read must repeat the same `--sdpsw_cntN` event numbers as the
> enable.** `--rd_sdpsw_perf` does NOT re-read the programmed event from the
> register unless you pass the cntN args; if you pass them, the printed name is
> driven by *your args*, not by what is actually programmed in hardware. So if the
> enable and the reads disagree, the labels will be wrong/misleading. Always keep
> enable and all reads on the identical event set, and confirm the enable line at
> the top of the log matches what you intended. (A real capture once enabled
> 47/49/50/52 while the intent was 3/5/69/71 — every dump was then the wrong
> edge/direction.)

## Naming: SION == SDPSW, CAKE == IFoE

Event names are written from the perspective of the SDP switch:

- **SION** = SDPSW (the SDP switch itself)
- **CAKE** = IFoE
- **DX** = DXS/DXM (the data-fabric-side originator/completer; see
  `dev-knowledge-mi450-gpu`)
- **CHAININ / CHAINOUT** = the switch's chain ports (switch-to-switch)

An event name `A_B_<Channel><Suffix>` means **A -> B**, that channel, on that
edge. E.g. `DX_SION_ReqVld` = DX→SION Req valid; `SION_CAKE_0_ReqCreditVld` =
SION→CAKE Req-credit valid.

## Event-number map (the pairs that matter for credit)

For each directed edge, the script exposes Vld (a message/“**used**” event) and
CreditVld (a credit grant — “**granted**” event) for each channel (Req,
MstrData, RdRsp, WrRsp). The credit-debug pairs are **Req** and **MstrData**:

### DXS -> SDPSW (`DX_SION_*`) — DF side, into the switch

| Event | Name | Meaning |
|------:|------|---------|
| 3 | `DX_SION_ReqVld` | Req used (DX→SION) |
| 5 | `DX_SION_ReqCreditVld` | Req credit granted |
| 6 | `DX_SION_MstrDataVld` | Data used |
| 8 | `DX_SION_MstrDataCreditVld` | Data credit granted |

(4 = ReqVld&ReqWr; 7 = MstrDataLast; 9-13 = RdRsp/WrRsp Vld/Last/CreditVld.)

### SDPSW -> IFoE (`SION_CAKE_*`) — switch out to IFoE

| Event | Name | Meaning |
|------:|------|---------|
| 69 | `SION_CAKE_ReqVld` | Req used (SION→CAKE) |
| 71 | `SION_CAKE_0_ReqCreditVld` | Req credit granted |
| 72 | `SION_CAKE_MstrDataVld` | Data used |
| 74 | `SION_CAKE_0_MstrDataCreditVld` | Data credit granted |

(70 = ReqVld&ReqWr; 73 = MstrDataLast; 75-79 = RdRsp/WrRsp.)

### Other edges (same pattern, for completeness)

- **SDPSW -> DXM** `SION_DX_*`: 47 ReqVld, 49 ReqCreditVld, 50 MstrDataVld,
  52 MstrDataCreditVld (53-57 RdRsp/WrRsp).
- **IFoE -> SDPSW** `CAKE_SION_*`: 25 ReqVld, 27 ReqCreditVld, 28 MstrDataVld,
  30 MstrDataCreditVld.
- **CHAININ** `CHAININ_SION_*` / `SION_CHAININ_*`: 14-24 / 58-68.
- **CHAINOUT** `CHAINOUT_SION_*` / `SION_CHAINOUT_*`: 36-46 / 80-90.
- 2 = `sdpsw_pg_iso_pulse`; 91-122 / 224-255 = `DEBUG_OUT[*]`.

> The full authoritative list is `SDPSW_COUNTERS` in the script — read it rather
> than trusting this summary if an event number is unfamiliar.

### Convention used for credit debug

To watch **request + data credit on one edge** in a single 4-slot capture, the
natural slot assignment is:

```
cnt0 = <edge>_ReqVld          # req used
cnt1 = <edge>_ReqCreditVld    # req granted
cnt2 = <edge>_MstrDataVld     # data used
cnt3 = <edge>_MstrDataCreditVld   # data granted
```

To instead compare **req credit across two edges** (e.g. DX→SION vs SION→CAKE),
put a (Vld, CreditVld) pair from each edge:
`cnt0/1 = DX_SION_ReqVld/ReqCreditVld`, `cnt2/3 = SION_CAKE_ReqVld/ReqCreditVld`
(= events 3 / 5 / 69 / 71).

## Interpreting the numbers

The read prints, per station, a block ending in four lines like:

```
----- MID1 SDP_SW0 Perf Counter Values -----
...
Counter0 - Event SION_DX_ReqVld: 51043280
Counter1 - Event SION_DX_0_ReqCreditVld: 51043343
Counter2 - Event SION_DX_MstrDataVld: 128915304
Counter3 - Event SION_DX_0_MstrDataCreditVld: 128915367
```

- The `Counter0..3 - Event <name>: <decimal>` summary lines hold the final 64-bit
  value (low + (upper<<32)); the preceding raw `READ ... PERF_COUNTN[_UPPER]`
  register lines are the halves.
- **used** = the `*Vld` count (messages the producer sent).
- **granted** = the matching `*CreditVld` count (credits the consumer issued).
- **held credit = granted - used** = credit currently outstanding (buffer space
  the consumer has advertised beyond what's been consumed). This is the number to
  watch.
- To isolate credit **issued at a connect** (initial credit), diff a counter
  against the previous dump: **net-granted-this-interval = Δgranted - Δused**. A
  reconnect that (re)issues initial credit shows a step here; steady streaming
  shows ~0.

### Station addressing

`get_sdpsw_instances` enumerates up to **18 SDP_SW stations per MID**. On
MI450/Helios the SerDes split is **MID0 -> CPU, MID1 -> IFoE** (see
`dev-knowledge-helios`), so for **IFoE** credit debug the active stations are the
**MID1 SDP_SW0..17** set. MID0 stations are generally not IFoE traffic; treat
nonzero MID0 counts as not-IFoE (e.g. MID0/SDP_SW13 is a known non-IFoE user).

### Expected IFoE initial credit

IFoE is expected to grant **14 Req credits** and **18 Data credits** per connect.
A connect happens on **first traffic** after a (re)connect, so the expected
signature is a **+14 req / +18 data** step in held credit (grant - used) on the
interval spanning that connect. When held-credit values do not match the expected
grant, the first thing to rule out is a capture-config mistake (wrong
edge/direction selected — see the gotcha under "The script") before reading
anything into the numbers.

## Quick tabulation recipe

Given a log with several dumps taken around successive operations, parse each
`----- MID<m> SDP_SW<n> Perf Counter Values -----` block, pull the four
`Counter<i> - Event <name>: <val>` lines, and emit per station/edge:
`used`, `granted`, `delta = granted - used`. Then a second view of
`Δgranted - Δused` between consecutive dumps surfaces where initial credit is
(re)issued. Restrict to MID1 for IFoE.
