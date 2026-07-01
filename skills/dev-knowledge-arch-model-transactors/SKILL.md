---
name: dev-knowledge-arch-model-transactors
description: >
  The IFoE ss-emu transactor layer in the arch model: ctctb (the "C to CTB" glue
  giving a C interface over the C++ CTB emulation transactors, for SDP and AXI),
  ctvved (the C wrapper over the Veloce VVED ethernet testbench API), and the
  emu_sdp / emu_eth wrappers that adapt them to the model's sdp_port / eth_port.
  Covers the CTCTB_XACTOR_DIRECT ("bypass") compile flag (raw per-beat SDP vs the
  TLM path), the SDP orig/cmpl beat API, the AXI register sideband
  (ifoe_*32_reg -> emu_zephyr -> ctctb_axi_*), the ctvved send-queue + Rx/Tx
  callback inversion, emu_eth preamble handling, and the chip-directory bodgering
  that frees the low-level transactors. Use when reading, writing, reviewing, or
  debugging ctctb, ctvved, emu_sdp, emu_eth, the SDP/AXI transactor path, the
  direct-vs-TLM beat handling, or the chip xtor overlay. For the runtime that uses
  this see dev-knowledge-ss-emu-environment; for the chip overlay branching see
  dev-knowledge-repo-ifoe-emu-overlays.
---

# IFoE ss-emu transactor layer (ctctb / ctvved / emu_sdp / emu_eth)

The emulation back end reaches three transactors — **SDP** and **AXI** (via
`ctctb`) and **ethernet** (via `ctvved`) — and the `emu_sdp` / `emu_eth` wrappers
adapt them to the model's port abstractions. The code is intricate; the detail
below is deliberately specific. Paths are under a workarea `M`; arch-model source
is `M/ifoe-arch-model/src/libs`.

The model presents its transactors as ops-vtable "symmetric ports" (`sdp_port_t`,
`eth_port_t`, via `SL_PORT_MAKE_SYM_TYPE`); that vtable machinery is out of scope
here beyond noting **emu provides one implementation** of each (`emu_sdp`,
`emu_eth`). This skill is about the emu implementations and the transactor glue
beneath them.

## ctctb — "C to CTB"

**ctctb = "C to CTB".** The CTB (chip test bench) emulation transactors are
**C++**; ctctb wraps them in a **C interface**, abstracting away the TLM-like
transactor machinery the IFoE code does not care about. It exposes:

- **SDP**: `ctctb_get_sdp_orig(ctctb, "<scope>")` /
  `ctctb_get_sdp_cmpl(ctctb, "<scope>")` — the originator and completer SDP
  channels.
- **AXI**: `ctctb_get_axi_master(ctctb, "<scope>")`, `ctctb_axi_read32`,
  `ctctb_axi_write32` (`include/ctctb/ctctb.h:83-88`).

It is also the IFoE end of the `ctb_cmd` interface: a `ctb_cmd_input` line is
dispatched (standard AMD CTB path) and ends, via `ctctb_ProcessCtbCommandImpl`,
at the `extern "C" ctctb_cmd_handler_callback` the **test app implements**,
resolved at link time when the test-app `.a` is linked into `libdpi.so`. (Runtime
picture: see `dev-knowledge-ss-emu-environment`.)

`ctctb.sbs` sets build-time C_DEFINEs:
`SOCAPI_CTB_CMD_IMPLEMENTATION_H = "socapi_ctb_cmd_impl.h"` (the header the SoCAPI
requestor `#include`s to reach ctctb), and `HWEMUL_SDP_ADP` / `HWEMUL_SDP_PARITY`
(SDP feature enables consumed by the SDP UVC transactor modules, not ctctb.cc).

### CTCTB_XACTOR_DIRECT — the "bypass" flag
There is **no literal "BYPASS" token and no env/runtime switch** (no `getenv`
anywhere) — every switch is compile-time. The flag the "bypass" idea refers to is
**`CTCTB_XACTOR_DIRECT`** (`include/ctctb/ctctb.h:12`, hard-coded `1`; change by
editing the header). It guards ~40 `#if` blocks and picks the SDP route:

- **`=1` DIRECT (current build):** ctctb talks **directly to the raw SDP UVC
  per-beat transactor channels**, bypassing the TLM / `cEmuSoc15HybridPath`
  gasket framework. Orig inherits `cEmuTransactionUser`, grabs the transactor
  scopes by name (`.req_drv`, …) and subscribes. API is per-beat:
  `alloc/free/put_req_beat`, `…_orig_data_beat`; callbacks deliver raw `*_beat_t`
  structs wrapping `*_cEmuTrans` objects. Parity is computed by hand.
- **`=0` non-DIRECT (dead here):** whole-transaction TLM path (`put_req` /
  `ReceiveTlm`, 512-bit payloads, split-response reassembly).

So "bypass" = **bypass the TLM abstraction, go straight to the SDP beat
transactors**. It is *not* SDP-vs-AXI and *not* a credit/protocol bypass; **AXI
is unaffected** (no DIRECT split on the AXI side).

Auxiliary compile flags (`ctctb.cc`): `IFOESW_185_WORKAROUND` (=1, per-request
side-table `m_reqs` to recover info the beat stream omits); `IFOESW_185_SDP_ORIG_RSP_CHAN`
(auto `0` when DIRECT); `IFOESW_186_SDP_ORIG_WR_RSP_ACC_ID` (=1, stash acc-ids for
write-response).

## emu_sdp — model sdp_port ↔ ctctb SDP beats

`emu_sdp_create(ctctb, sched, sdp_orig, sdp_cmpl)` (`emu_sdp/emu_sdp.c:150`) wires
`emu_sdp_port_orig_ops` iff `sdp_orig!=NULL` and `emu_sdp_port_cmpl_ops` iff
`sdp_cmpl!=NULL`, registers ctctb callbacks, and connects. Called once at
`ifoe_ss_model_emu.c:87` with `ctctb_get_sdp_orig(ctctb,"orig_sdp0_rc")` and
`ctctb_get_sdp_cmpl(ctctb,"cmpl_sdp0_rc")`. `emu_sdp_get_port` hands out the
embedded `sdp_port_t`.

**orig vs cmpl (note the naming):** the ctctb **`orig`** channel carries requests
the *model originates* and their responses; the ctctb **`cmpl`** channel receives
requests *targeting the model* and sends the model's responses back. So
model-as-originator → `orig`; model-as-completer → `cmpl`.

Request path (model → beats, DIRECT), `emu_sdp_orig_push_request`
(`emu_sdp.c:711`):
```
req_beat = ctctb_sdp_orig_alloc_req_beat(sdp_orig);
org_request_msg_to_req_beat(request, req_beat);      // + parity
ctctb_sdp_orig_put_req_beat(sdp_orig, req_beat);
ctctb_sdp_orig_free_req_beat(...);
```
Write data uses `…_orig_data_beat` analogues. Response path (ctctb beat callback →
model): the callback converts the beat to a msg and defers onto the sim scheduler
(`sched_pend_event_now`) which calls `sdp_port_cmpl_egress_read_response(...)` (so
the VVED/transactor callback thread hands off to the sim thread). Completer inbound
(`emu_sdp_cmpl_push_request` / `_push_data`) pushes onto per-side FIFOs and
`emu_sdp_cmpl_dispatch_request_data` re-serialises request-before-data ordering
before egressing to the model; model responses go back via
`ctctb_sdp_cmpl_put_rd_rsp_beat` / `…_wr_rsp_beat`.

`#if CTCTB_XACTOR_DIRECT` substantially changes emu_sdp too: DIRECT uses per-beat
structs + manual parity + the FIFO/dispatch reordering; non-DIRECT accumulates
whole transactions with beat/length conversions and a split-response state
machine. Struct fields and callback signatures switch on the flag.

## AXI register sideband (separate from SDP)

Firmware/model register access reaches the AXI master through ctctb, via
`emu_zephyr` (which implements Zephyr's `sys_read32`/`sys_write32`):
```
ifoe_write32_reg(...)                 external/mpifoe-fw/.../ifoe_drv/ifoe_regs.h:71
  -> ifoe_write32(...) { sys_write32(data, base+reg); }        ifoe_regs.h:63
    -> sys_write32(...) { ctctb_axi_write32(state.axim, ...); } emu_zephyr/zephyr.c:70
```
Reads mirror it. The AXI master handle is fetched once
(`ctctb_get_axi_master(ctctb,"vulcano_axi_mst")` in
`ifoe_ss_model_emu_wrap.c:27`) and passed to `emu_zephyr_init`. Note
`ifoe_sdp_shim_drv_emu` is **not** the AXI bridge — it merely *uses*
`ifoe_*32_reg` on SHIM SCRATCH regs, riding the same emu_zephyr→`ctctb_axi_*`
path.

## ctvved — C wrapper over the VVED ethernet testbench API

`ctvved` wraps the Veloce **VVED** (Veloce Virtual Ethernet Device) testbench
library `libVE_TestBench_API`. `ctvved_dummy` is a no-op stand-in stubbing the
same symbols (for builds without VVED).

- **Lifecycle:** `ctvved_init` spawns a poll thread that waits on
  `IsMACAvailable(mac)` for MACs 1..4; on availability it inits the port,
  registers the send-side callback, reads PHY config, and fires the caller's
  `port_avail_cb`. Config goes via `SendTclCommand` / `SendPythonCommand`;
  start sends `start … --testbenchGeneration --enableTBSnoop true`.
- **Callback inversion (confirmed):** VVED's **Rx** callback is our **send** path
  — `RegisterRxCallbackFunction(mac, RxPacketCallbackFunction, port, true)`;
  `RxPacketCallbackFunction` *drains our send queue* (VVED pulls from us to
  transmit). VVED's **Tx** callback is our **receive** path —
  `RegisterTxCallbackFunction(mac, TxPacketCallbackFunction, port)`;
  `TxPacketCallbackFunction` calls our `recv_cb` with an inbound packet.
- **Upward C API:** `ctvved_eth_get_port(ctvved, mac)`,
  `ctvved_eth_send(port, data, len)`, `ctvved_eth_set_recv_cb(port, cb, ctx)`,
  recv type `void ctvved_eth_recv_cb(bool is_packet, const uint8_t *data,
  size_t len, void *cbctx)`.
- **Send queue:** `ctvved_eth_send` copies the buffer and pushes onto
  `pkt_queue` under a mutex; VVED's Rx callback pops one per call (setting CRC /
  IFG options).

## emu_eth — model eth_port ↔ ctvved

`emu_eth_create(ctvved_port, sched)` (`emu_eth/emu_eth.c:29`) registers
`emu_eth_ctvved_recv` on the ctvved port and presents an `eth_port_t` whose
`ops.ingress = emu_eth_ingress`. FCS is compile-gated off (calc_crc=false);
preamble handling is on (`CONFIG_EMU_ETH_HANDLE_PREAMBLE`), preamble =
`{0xfb,0x55,0x55,0x55,0x55,0x55,0x55,0xd5}`.

- **TX (app → wire):** `emu_eth_ingress` prepends the 8-byte preamble, then
  `ctvved_eth_send(...)`.
- **RX (wire → app):** `emu_eth_ctvved_recv` drops IFG (`!is_packet`), asserts &
  strips the 8-byte preamble, copies into a fresh `eth_pkt_t`, and defers via
  `sched_pend_event_now` → `eth_port_egress` (hopping from the VVED callback
  thread to the sim scheduler thread).

Full chains:
- **TX:** `eth_port` ingress → `emu_eth_ingress` (prepend preamble) →
  `ctvved_eth_send` (queue) → VVED **Rx** cb (pop) → emulated MAC.
- **RX:** emulated MAC → VVED **Tx** cb → `emu_eth_ctvved_recv` (strip preamble,
  drop IFG) → `sched_pend_event_now` → `eth_port_egress` → model.

## Chip-directory bodgering (frees the low-level transactors)

For ctctb's DIRECT path to own the SDP datapath, the released chip emulation model
must stop driving it. IFoE's overlay commits on the chip per-release branch do
this (above the `.extras/import.sh …` import commit; see
`dev-knowledge-repo-ifoe-emu-overlays`). Load-bearing groups:

- **Suppress released datapath / port-control** (under
  `chip/emu_rtl/src/bfms/amd_sdp_uvc/ctb/`): `IFOESW-231` `#if 0`s the **monitor**
  registrations in the SDP orig/cmpl TLM gaskets (the released observers that
  would parse/forward beats), leaving IFoE's path authoritative (drivers left
  intact); `IFOESW-464` comments out `DoSdpPortControlProtocol(...)` (released SDP
  connect/disconnect handshake).
- **Wire ctctb in as the CTB:** `IFOESW-88` (compile ctctb via
  `$ENV{CTCTB_HOME}/ctctb.sbs` in place of the bundled example); `IFOESW-158`
  (stop SocApi touching the simnow/ctb queues); `IFOESW-147` (atomic
  `latest_timestamp` for ctctb's threading).
- **SDP beat-field correctness:** gasket parity/VC/ReqNumBeats fixes.
- **Scaffolding:** `ifoe_emu_test_app`; `IFOESW-410` feeds a libdpi `port_file`
  into the `socapi ifoe_init $port` command.

In one line: **chip bodgering frees the low-level SDP transactors and installs
ctctb as the CTB; ctctb (DIRECT) then drives them per-beat, emu_sdp/emu_eth adapt
to the model ports, and the AXI sideband rides emu_zephyr→ctctb_axi_*.**

> Confidence: ctctb flags, emu_sdp/emu_eth/ctvved chains, the AXI path, and the
> chip overlay commits are VERIFIED from source/git. One link is INFERRED: that
> ctctb (DIRECT) is the consumer of the datapath freed by the chip
> monitor-suppression — the build wiring confirms ctctb is installed as the CTB,
> but the end-to-end runtime hand-off was not traced line-by-line.

## Cross-references

- **The model flavour above this glue** (the emu `ifoe_ss_model_t`, alongside the
  arch/cmod flavours): `repo-ifoe-arch-model-ss-model`
- **Runtime that uses this**: `dev-knowledge-ss-emu-environment`
- **chip/_ip/_env overlay branch model**: `dev-knowledge-repo-ifoe-emu-overlays`
- **Build**: `build-ifoe-arch-model-ss-emu`
- **arch-model repo overview**: `repo-ifoe-arch-model`
