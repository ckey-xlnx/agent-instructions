---
name: dev-knowledge-ifoe-sdp-registers
description: >
  Reference for the register / programming-model interface the firmware uses to
  drive the IFoE SDP hardware — the memory-mapped control and status registers
  for the two SDP ports (COMP facing DXS, ORIG facing DXM), how the hardware
  brings each port up and tears it down, and the connect/disconnect programming
  model. Distinct from the wire-level SDP protocol and the perf-counter debug
  tooling. Use when reading, writing, reviewing, or debugging firmware that
  programs the IFoE SDP block: the connect-control / connect-status /
  orig-activity / orig-disconnect registers, the COMP-port mirror behaviour, the
  ORIG-port TX-only asymmetry, the race-free idle disconnect handshake, or SDP
  connect/quiesce. For the abstract SDP link protocol (Originator/Completer
  roles, *ClkReq/*ClkAck signals, credit model) see [[dev-knowledge-sdp-interface]];
  for the IFoE core block architecture see [[dev-knowledge-ifoe-core]]; for SDP
  credit perf-counter debug see [[dev-knowledge-sdp-credit-counters]].
  Trigger phrases: "IFoE SDP register", "IFoE register interface", "SDP register
  map", "ifoe_drv", "cfg_sdp_connect_control", "sta_sdp_connect_status",
  "cfg_sdp_orig_activity", "cfg_sdp_orig_disconnect", "SDP connect control",
  "force_disconnect", "force_connect", "ORIG port", "COMP port", "DXS connect",
  "DXM connect", "program the SDP connect", "SDP status register".
---

# IFoE SDP Register Interface

The software-facing view of the IFoE SDP hardware: the memory-mapped registers
firmware reads and writes to connect, observe, and disconnect the two SDP ports,
plus a model of what the hardware does in response. This is distinct from:

- the **wire-level protocol** ([[dev-knowledge-sdp-interface]]) — the
  Originator/Completer roles, the `*ClkReq`/`*ClkAck` handshake and the credit
  model that these registers ultimately drive, and
- the **debug perf counters** ([[dev-knowledge-sdp-credit-counters]]) — read-only
  observability tooling, not the control interface.

## Confidence / provenance

Read this before trusting any signal name below.

- **Authoritative** (PPR register pages + hwdefs header
  `src/drivers/hwdefs/lsc/ifoe/include/hwdefs/ifoe.h` + driver
  `src/drivers/ifoe_drv/ifoe_common.c`): every register **address, bit position,
  field name, access type, and the PPR field description**.
- **Reconstructed model** (NOT in any register doc): the mapping of register bits
  to specific SDP `*ClkReq`/`*ClkAck` signals, and the per-cycle hardware
  state-machine pseudocode below. This is inferred by joining the register
  descriptions to the SDP protocol and to observed bring-up behaviour. Treat it
  as a working model, not silicon truth; correct it against RTL/observation when
  they disagree.

## The two ports

IFoE presents two SDP ports, and its **role differs on each**, so the *same*
signal name denotes a different physical driver on each port:

| Port | Faces | IFoE role | Device role | Driver split |
|------|-------|-----------|-------------|--------------|
| **COMP** | DXS | Completer | Originator | DXS drives `Orig*`, IFoE drives `Comp*` |
| **ORIG** | DXM | Originator | Completer | IFoE drives `Orig*`, DXM drives `Comp*` |

(The port↔device mapping DXS=COMP, DXM=ORIG is authoritative — it is how the
driver's `dxs_*` vs `dxm_*` helpers read the status bits.)

Each port has **two independently-gated directions**. Throughout, **TX =
IFoE→Device, RX = Device→IFoE**. A direction comes up when one side asserts its
`*ClkReq` and the other answers with `*ClkAck`:

```
COMP port:
   DXS->IFoE  (COMP RX):  DXS  asserts COMP.OrigClkReq  +  IFoE asserts COMP.CompClkAck
   IFoE->DXS  (COMP TX):  IFoE asserts COMP.CompClkReq  +  DXS  asserts COMP.OrigClkAck
ORIG port:
   IFoE->DXM  (ORIG TX):  IFoE asserts ORIG.OrigClkReq  +  DXM  asserts ORIG.CompClkAck
   DXM->IFoE  (ORIG RX):  DXM  asserts ORIG.CompClkReq  +  IFoE asserts ORIG.OrigClkAck
```

> PPR spells COMP as "Comparator" in the status-bit descriptions. That is a doc
> quirk; in SDP terms **COMP = Completer**.

## Register map (live fields only)

All offsets are within the IFOECOMMON CSR map; per-instance SMN base e.g. IFOE0 =
`0x8C0C0000`, so IFOE0 `cfg_sdp_connect_control` = `0x8C0C8224`. Captured /
sample status bits (the non-`_now` variants) are intentionally omitted — only the
live bits matter for control.

### `cfg_sdp_connect_control` @ `0x8224` — RW

What IFoE is permitted (or forced) to drive. Reset 0.

| Bit | Field | Effect (reconstructed signal) |
|----:|-------|-------------------------------|
| 0 | `orig_connect_accept` | permit IFoE to assert `ORIG.OrigClkAck` (ack DXM's `CompClkReq`) |
| 1 | `comp_connect_accept` | permit IFoE to assert `COMP.CompClkAck` (ack DXS's `OrigClkReq`) |
| 2 | `orig_connect_enable` | permit IFoE to assert `ORIG.OrigClkReq` when it has traffic |
| 3 | `comp_connect_enable` | permit IFoE to assert `COMP.CompClkReq` |
| 4 | `orig_force_connect`  | IFoE asserts `ORIG.OrigClkReq` *without* traffic (needs accept+enable) |
| 5 | `comp_force_connect`  | IFoE asserts `COMP.CompClkReq` *without* traffic (needs accept+enable) |

### `sta_sdp_connect_status` @ `0x8228` — RO

Live link state. Reset 0. (Bits 2,3,6,7 are the captured-at-interrupt variants —
omitted.)

| Bit | Field | Meaning |
|----:|-------|---------|
| 0 | `orig_connect_state` | ORIG up in **both** directions (= TX_now & RX_now) |
| 1 | `comp_connect_state` | COMP up in **both** directions |
| 4 | `orig_rx_state_now` | DXM→IFoE live |
| 5 | `orig_tx_state_now` | IFoE→DXM live |
| 8 | `comp_rx_state_now` | DXS→IFoE live |
| 9 | `comp_tx_state_now` | IFoE→DXS live |

### `cfg_sdp_orig_activity` @ `0x822c` — RO

ORIG-port (IFoE-as-originator) busy-ness. Reset 0.

| Bit | Field | Meaning |
|----:|-------|---------|
| 31 | `reqs_pending` | responses still outstanding (not yet received) |
| 30 | `buf_not_empty` | origreq or origdata output buffers non-empty |
| 29:0 | `activity_count` | increments when an orig **req** is written to the SDP output buffer (req only) |

Fully idle ⇔ `!buf_not_empty && !reqs_pending`. The count is **not** the idle
test — it counts requests only (not responses); it exists purely as the race
token for the disconnect handshake below.

### `cfg_sdp_orig_disconnect` @ `0x8230` — RW

ORIG-port disconnect. **There is no COMP equivalent.** Reset 0.

| Bit | Field | Meaning |
|----:|-------|---------|
| 30 | `force_disconnect` | RW, **self-clearing**. Disconnect ORIG even if the count differs. |
| 29:0 | `activity_count` | write back the value last read from `0x822c`. HW disconnects only if its live count *still equals* this AND `force_disconnect`=0 (race-free idle confirm). |

## Hardware model (per port, per cycle)

Reconstructed — see *Confidence* above. This is what makes the two asymmetries
self-evident.

```
# CC = cfg_sdp_connect_control(0x8224)
# status bits = sta_sdp_connect_status(0x8228); ACT = cfg_sdp_orig_activity(0x822c)

# ===== COMP port (faces DXS; IFoE = Completer) =====
on COMP port each cycle:
    # RX half: DXS->IFoE
    if DXS asserts COMP.OrigClkReq and CC.comp_connect_accept:
        IFoE asserts COMP.CompClkAck
        comp_rx_state_now = 1
    if DXS deasserts COMP.OrigClkReq:
        IFoE deasserts COMP.CompClkAck
        comp_rx_state_now = 0
    # UNKNOWN: comp_connect_accept cleared while comp_rx_state_now=1 -- unobserved;
    #   not asserting whether CompClkAck drops here

    # TX half: IFoE->DXS  -- the observed "mirror": IFoE follows the RX DXS opened
    if (comp_rx_state_now and CC.comp_connect_enable) or CC.comp_force_connect:
        IFoE asserts COMP.CompClkReq
    else if not comp_rx_state_now and not CC.comp_force_connect:
        IFoE deasserts COMP.CompClkReq   # mirror collapses once the RX it followed drops
    # UNKNOWN: comp_connect_enable cleared while comp_tx_state_now=1 -- unobserved;
    #   not asserting whether CompClkReq drops here
    if DXS asserts COMP.OrigClkAck:
        comp_tx_state_now = 1

    comp_connect_state = comp_rx_state_now and comp_tx_state_now

# ===== ORIG port (faces DXM; IFoE = Originator) =====
on ORIG port each cycle:
    # TX half: IFoE->DXM  -- IFoE originates on traffic OR force
    have_traffic = ACT.buf_not_empty
    if (CC.orig_connect_enable and have_traffic) or CC.orig_force_connect:
        IFoE asserts ORIG.OrigClkReq
    # NB: OrigClkReq is NOT dropped when have_traffic goes low -- once up it stays
    #   asserted until torn down via cfg_sdp_orig_disconnect (see below)
    # UNKNOWN: orig_connect_enable cleared while orig_tx_state_now=1 -- unobserved;
    #   not asserting whether OrigClkReq drops here
    if DXM asserts ORIG.CompClkAck:
        orig_tx_state_now = 1

    # RX half: DXM->IFoE  -- DXM originates the reverse path when it has responses
    if DXM asserts ORIG.CompClkReq and CC.orig_connect_accept:
        IFoE asserts ORIG.OrigClkAck
        orig_rx_state_now = 1
    if DXM deasserts ORIG.CompClkReq:
        IFoE deasserts ORIG.OrigClkAck   # ack follows req, mirror of COMP RX half
        orig_rx_state_now = 0
    # UNKNOWN: orig_connect_accept cleared while orig_rx_state_now=1 -- unobserved;
    #   not asserting whether OrigClkAck drops here

    orig_connect_state = orig_tx_state_now and orig_rx_state_now

    # ORIG disconnect: count-confirm handshake (HW side)
    on write to cfg_sdp_orig_disconnect(0x8230):
        if force_disconnect or (written.activity_count == live ACT.activity_count):
            IFoE deasserts ORIG.OrigClkReq -> orig_tx_state_now = 0
        # else a req raced in; counts differ -> ignore, stay connected
    # UNKNOWN: interaction with orig_force_connect -- if force_connect is still
    #   set, the TX-half logic above would re-assert OrigClkReq next cycle.
    #   Whether disconnect checks !orig_force_connect first is unobserved.
    # No idle timer / no autonomous teardown in this register set.
```

## Corners

1. **COMP bring-up is DXS-driven; IFoE mirrors.** With accept+enable armed, the
   sequence is: DXS `OrigClkReq` → IFoE `CompClkAck` (RX up) → IFoE *mirrors* with
   `CompClkReq` → DXS `OrigClkAck` (TX up) → `comp_connect_state`=1. In normal
   operation IFoE follows rather than initiating COMP; the exception is
   `comp_force_connect`, which makes IFoE assert `CompClkReq` on its own.

2. **ORIG settles TX-only.** After passing traffic to DXM you get
   `orig_tx_state_now`=1 but `orig_rx_state_now`=0, so `orig_connect_state`=0 even
   though traffic is flowing. Expected: nothing makes the completer (DXM) open the
   reverse path, so IFoE's `orig_connect_accept` has nothing to answer.

3. **No hardware auto-disconnect in scope.** These four registers expose only
   FW-initiated ORIG disconnect (count-confirm or force). No bit here makes HW
   drop a port on idle by itself. (Stated from absence in this register set — not
   proof the silicon never does it via some other block.)

4. **Disconnect is ORIG-only.** There is no `cfg_sdp_comp_disconnect` and no COMP
   activity register. Tearing COMP down from the IFoE side — if it is even
   possible — could only go via clearing `comp_connect_enable`/`accept`, and that
   is **unverified**.

5. **Force reconnect = set-then-clear.** Observed: setting `orig_force_connect`
   then clearing it forces IFoE→DXM back up (accept+enable must already be set).

## Driver entry points (as of writing — verify before citing)

In `src/drivers/ifoe_drv/ifoe_common.c`:

- `ifoe_common_set_showtime` — arms all four accept+enable bits, clears force.
- `ifoe_common_dxm_idle` — reads `cfg_sdp_orig_activity` (`buf_not_empty`,
  `reqs_pending`).
- `ifoe_common_dxm_disconnect` — clears `orig_connect_enable`.
- `ifoe_common_dxs_disconnected` / `ifoe_common_dxm_disconnected` — read the
  `*_state_now` bits of `sta_sdp_connect_status`.
