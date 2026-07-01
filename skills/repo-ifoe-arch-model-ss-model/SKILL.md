---
name: repo-ifoe-arch-model-ss-model
description: >
  The ifoe_ss_model_t abstraction in the ifoe-arch-model repo: the single model
  interface (config ops, sdp_port_t, eth_port_t) and its THREE interchangeable
  flavours below that interface — arch (pure software model), emu (subsystem
  emulation, via ctctb/ctvved transactors), and cmod (the NTSG model as used in
  SimNow). Covers what each flavour does beneath the interface line: the config
  path (real mpifoe-fw drivers vs faked drivers) and the port transport
  (transactor vs direct call into the model). Use when reading, writing, or
  debugging ifoe_ss_model_t, ifoe_instance_create/get_sdp_port/get_eth_port, the
  arch vs emu vs cmod model implementations, the fw driver split, or "how is the
  model the same across emulation / SimNow / pure arch". This is the model
  INTERNALS (below the interface); how a harness WIRES several instances together
  and drives them is variant-specific — see dev-knowledge-ss-emu-selfcontained-app
  (and the TE flow). For the beat-level transactor glue beneath the emu flavour
  see dev-knowledge-arch-model-transactors. Part of the ifoe-arch-model repo
  knowledge (see repo-ifoe-arch-model).
---

# ifoe_ss_model_t — the model interface and its flavours

`ifoe_ss_model_t` is the composable unit of the IFoE architectural model. A
harness creates one or more instances and drives them entirely through a small
generic interface; **below that interface line the instance can be any of three
flavours**, and the harness does not care which. This skill is about that
interface and what each flavour does beneath it.

**Scope: model internals only (below the interface line).** How a particular
harness *wires several instances together and runs them* (switches, test-app
drivers, binaries, processes) is above the line and lives with the variant — see
`dev-knowledge-ss-emu-selfcontained-app` for the self-contained ss-emu wiring,
and the TE flow for TE. The composability is exactly why the split is drawn
here: the same flavours, config path, and SDP/eth interfaces are reused
unchanged across ss-emu and TE; only the above-the-line wiring differs.

Paths below are under `src/libs` in the ifoe-arch-model repo.

## The interface (what every flavour presents)

Declared in `ifoe_ss_model/include/ifoe_ss_model/ifoe_interface.h`:

- **The instance**: `ifoe_ss_model_t` (`:40`), with
  `ifoe_instance_create` / `ifoe_instance_init` / `ifoe_instance_destroy`
  (`:45,74,51`).
- **SDP port**: `ifoe_instance_get_sdp_port(inst)` → `sdp_port_t *` (`:72`).
- **Ethernet ports**: `ifoe_instance_get_eth_port(inst, port_num)` → `eth_port_t *`
  (`:68`), with `ifoe_instance_get_num_eth_port` (`:59`).
- **Config ops**: the `ifoe_cfg_interface.h` set (`ifoe_config_set` /
  `_get`, `ifoe_configtime` / `ifoe_showtime`, network config, …).

`sdp_port_t` and `eth_port_t` are the generic "symmetric port" interface objects
(vtable pattern) shared across the model; their definitions and the beat/packet
semantics are documented with the transactor layer
(`dev-knowledge-arch-model-transactors`). Note one shape detail used below:
**`sdp_port_t` is a single struct carrying TWO ops-tables** — `_orig_ops` and
`_cmpl_ops` (`sdp/include/sdp/sdp_port.h:35-58`) — not two separate types.

> A per-flavour factory function, `ifoe_ss_model_init(...)`
> (`ifoe_ss_model_wrap.h`), stands up the sim environment and returns instances;
> which flavour you get is selected at link time by which `*_wrap` you link. It
> is just a factory — not part of the model interface — and does not affect the
> picture below.

## The two axes that distinguish the flavours

Every flavour differs on exactly two things beneath the interface:

1. **Config path** — how the config ops reach hardware state: through the **real
   mpifoe-fw drivers**, or through **faked drivers** that poke model state.
2. **Port transport** — how `sdp_port_t` / `eth_port_t` traffic leaves the model:
   through an **emulation transactor**, or as a **direct function call** into the
   model.

| flavour | config drivers | register access | sdp_port / eth_port transport |
|---|---|---|---|
| **arch** | faked `{.impl}` | none (no registers) | direct calls into arch stages / `xrmac` |
| **emu**  | real mpifoe-fw `{.base}` | zephyr → **ctctb AXI** | **ctctb** (SDP) / **ctvved** (eth) |
| **cmod** (NTSG/SimNow) | real mpifoe-fw `{.base}` | zephyr → **direct `read_reg`/`write_reg`** | direct calls into NTSG model (`fake_xrmac`) |

The **fw driver split is by struct definition**, and it is the whole trick.
Both real-driver flavours and the faked flavour use the *same* driver type names
— `ifoe_t`, `xrpfc_t`, `xrmac_t`, `xrsec_t`, `ex_t`, bundled in `ifoe_fw_drvs_t`
(`ifoe_fw_drv_intf/include/ifoe_fw_drv_intf/ifoe_fw_drv_intf.h:9-15`) — and the
same FW subsystem layer (`external/mpifoe-fw/src/subsystems/ifoe_ss`, reached via
`ifoe_fw/ifoe_fw.c:19`). Only the driver *struct definition* differs per flavour.

## arch — pure software model

`ifoe_ss_model_arch` (`ifoe_ss_model_arch/ifoe.c`).

- **SDP**: the model implements `sdp_port_orig_ops` / `sdp_port_cmpl_ops`
  directly (`ifoe.c:85-95`); ingress/egress call into the arch pipeline stages
  (rx/tx stages, retag, sdp_shim).
- **Config / drivers**: the driver structs are back-pointers into the model —
  `struct ifoe_s { ifoe_impl_t *impl; }` etc. (`ifoe.h:79-96`, comment: "All the
  state for our fake drivers is in the model context"), built at `ifoe.c:140-156`.
  The driver bodies live in `ifoe_drv_interface.c` and **poke model state
  directly** — there are no `sys_read32`/`sys_write32` and no `ctctb` calls.
- **Ethernet**: `eth_port_t` per port from the model (`ifoe.c:368-376`); the MAC
  is `xrmac` (`xrmac.c:138` `xrmac_ops`, `xrmac_ingress`).

## emu — subsystem emulation

`ifoe_ss_model_emu` (`ifoe_ss_model_emu/ifoe_ss_model_emu.c`).

- **SDP**: `emu_sdp` implements the port and drives the emulation SDP beat
  transactors via **ctctb** — created with the orig/cmpl scopes
  `ctctb_get_sdp_orig(ctctb,"orig_sdp0_rc")` / `_cmpl(…"cmpl_sdp0_rc")`
  (`:87-89`); `ifoe_instance_get_sdp_port` returns `emu_sdp_get_port` (`:124-127`).
- **Config / drivers**: real mpifoe-fw drivers, struct definitions carrying base
  addresses — `{.base = 0xc0000}` etc. (`ifoe_fw_drv/ifoe_fw_drv.c:16-30`). The
  real driver does register I/O: `ifoe_write32` → `sys_write32`
  (`external/mpifoe-fw/.../ifoe_drv/ifoe_regs.h`) → `ctctb_axi_write32`
  (`emu_zephyr/zephyr.c:71`); `emu_zephyr` is wired to the AXI master
  `ctctb_get_axi_master(ctctb,"vulcano_axi_mst")`.
- **Ethernet**: `emu_eth` per port (`:82-83`), built on **ctvved** (the C wrapper
  over the Veloce VVED testbench API); `ifoe_instance_get_eth_port` returns
  `emu_eth_get_port` (`:116-121`).

The beat-level detail of ctctb / emu_sdp / ctvved / emu_eth (the DIRECT bypass
flag, parity, the Rx/Tx callback inversion, preamble handling) is owned by
`dev-knowledge-arch-model-transactors` — do not duplicate it here.

## cmod — the NTSG model (as used in SimNow)

`ifoe_ss_model_cmod` (`ifoe_ss_model_cmod/`, C++). Wraps the **NTSG** model used
in SimNow. It sits between emu and arch on the two axes: **real registers like
emu, direct-call ports like arch.**

- **Same interface**: implements `ifoe_instance_create` / `_get_sdp_port` /
  `_get_eth_port` (`ifoe_instance.cpp:33-75`).
- **Config / drivers**: real mpifoe-fw drivers (`ifoe_fw_drvs_get(0,&drvs)`,
  `ifoe_ss_model_cmod.cpp:76`) over its own zephyr shim `cmod_zephyr.cpp`
  (`sys_read32`/`sys_write32` at `:59,67`). But register access is a **direct
  call into the model** — `m_ifoe_cmod->read_reg` / `write_reg`
  (`ifoe_ss_model_cmod.cpp:332,340`) — **not** a transactor/AXI path.
- **Ports**: no transactors at all (no ctctb/ctvved). The eth MAC is
  `fake_xrmac` (`class fake_xrmac : public eth_port_t`, `fake_xrmac.h:35`),
  calling straight into the NTSG model (`step_network_pkt` / `get_network_pkt`).

## Diagram (below the interface line)

```
 HARNESS (any) drives instances only through:
    config ops        sdp_port_t (_orig_ops/_cmpl_ops)      eth_port_t
 ───────────────────────────── interface line ─────────────────────────────
   arch                       emu                        cmod (NTSG/SimNow)
   ────                       ───                        ──────────────────
 cfg: faked drivers        cfg: real fw drivers       cfg: real fw drivers
      {.impl}→model             {.base}                    {.base}
                                zephyr→ctctb_axi (AXI)     cmod_zephyr→read_reg/
                                                           write_reg (direct)
 sdp: direct → arch        sdp: emu_sdp → ctctb        sdp: direct → NTSG model
      pipeline stages           (orig/cmpl beats)
 eth: direct → xrmac       eth: emu_eth → ctvved       eth: fake_xrmac → NTSG
                                (VVED)                       model
        │                         │                            │
        ▼                         ▼                            ▼
   arch model state        Veloce emulator (RTL)         NTSG cmodel state
                           via SDP/AXI + VVED
```

## Cross-references

- **Beat-level transactor glue beneath the emu flavour** (ctctb / ctvved /
  emu_sdp / emu_eth, DIRECT flag, callback inversion):
  `dev-knowledge-arch-model-transactors`.
- **Above-the-line wiring for the self-contained ss-emu variant** (binaries,
  shared test/rdwr code, the switch): `dev-knowledge-ss-emu-selfcontained-app`.
- **Repo overview**: `repo-ifoe-arch-model`.
- **Building the model on ss-emu**: `build-ifoe-arch-model-ss-emu`.
