---
name: dev-knowledge-ss-emu-selfcontained-app
description: >
  The self-contained test-app variant of IFoE subsystem-emulation (ss-emu) — the
  ifoe_test flow that drives the model via ctb_cmd, using emu_eth_to_arch as the
  epgm app, launched with velocetool-ifoe. Covers how to form the velocetool
  command line for THIS variant: which structural args select it, and why
  --skip-rtl-rsync is required for the overlay workspace. Use when working on or
  running the self-contained ss-emu test app (ifoe_test), as opposed to the TE
  variant. For velocetool and its generic args see dev-knowledge-velocetool; for
  the grid job and results see infra-grid-atl / dev-knowledge-ss-emu-environment.
---

# Self-contained ss-emu test app (ifoe_test)

The **self-contained** ss-emu variant runs the IFoE arch model as a standalone
emulation test, driven through `ctb_cmd`, with `emu_eth_to_arch` as the epgm app.
This is the `ifoe_test` work area, as opposed to the TE variant.

## How this variant wires the models together

This is the *above-the-interface* wiring for this variant: how it composes
`ifoe_ss_model_t` instances and drives them. The model internals below the
interface (the arch/emu/cmod flavours, the config path, the SDP/eth port
implementations) are **not** here — they are owned by
`repo-ifoe-arch-model-ss-model`. This section is only what this variant does on
top.

Two counts appear below and are distinct: **N** = number of arch model instances
(`LOCAL_ACCELERATOR_COUNT`); **P** = number of IFoE ports per model
(`WITH_NUM_IFOE_PORTS` / `NUM_PORTS`).

### Two binaries, two roles

The run is **two separate binaries**, each holding its own model instance(s),
coupled **only over ethernet**:

- **`libdpi.so`** — loaded into the primary Veloce process. Holds **one `emu`
  flavour** `ifoe_ss_model_t` (the emulated DUT). Built per
  `build-ifoe-arch-model-ss-emu`; links `libifoe_ss_model_emu*` + real
  `libctctb` (`src/libs/emu_libdpi/Makefile`).
- **`emu_eth_to_arch`** — the epgm app on the VVED/EPGM host. Its `main()`
  (`src/apps/emu_eth_to_arch/main.c:96`) creates **N `arch` flavour** instances
  (via `ifoe_ss_model_init(N, …)` resolving to `ifoe_ss_model_arch_wrap`,
  `main.c:98-103`) — the software reference copies.

Both binaries run the **same test/stimulus code** against the generic model
interface; only the flavour behind the interface differs (emu vs arch). The
shared code is three libraries:

- **`ifoe_push_config`** (`ifoe_emu_temp_bringup_config`) — configures a model
  via its config ops (`main.c:154`, and in libdpi `ifoe_emu_test_app.c:84`).
- **`sdp_ex_test`** (`sdp_ex_test_wrap`) — the SDP exerciser bound to a model's
  `sdp_port_t` (`main.c:163`, `ifoe_emu_test_app.c:76`).
- **`ifoe_emu_test_rdwr`** — injects/handles read+write flows over that SDP
  exerciser (`_create` / `_add_flow` / `_start`; `main.c:148,166`,
  `ifoe_emu_test_app.c:94`).

### `ifoe_emu_test_app` is the libdpi-side driver

`ifoe_emu_test_app` (`src/libs/ifoe_emu_test_app/ifoe_emu_test_app.c`) is the
**libdpi-only wrapper** around that shared code. Its real job is to implement
`ctctb_cmd_handler_callback` (`:29`) so `ctb_cmd` lines drive the test:
`ifoe_init` builds one emu model + sdp_ex_test + pushes config + adds rdwr flows
(`:37,66-105`), `start` runs the rdwr sequence (`:46,119`), `ifoe_fini` joins
(`:42`). (The `ctb_cmd` mechanism itself is in
`dev-knowledge-ss-emu-environment`.) `emu_eth_to_arch` has its own `main()` and
calls the same three libraries directly.

### The ethernet coupling: a per-port switch

`emu_eth_to_arch` connects everything on the ethernet side (its `main.c`):

- It defines a **trivial `eth_switch`** locally (`main.c:24-71`) — an array of
  `eth_port_t` with a `get_dest` that routes on the accelerator id in the packet
  (`main.c:82-86`). It is **not** a library.
- It creates **P switches, one per IFoE port** (`main.c:112,141`).
- For each of the P ports it creates an **`emu_eth`** backed by the **real
  `ctvved`** (→ VVED) and connects it into that port's switch (`main.c:139-143`).
- For each of the N arch models it connects that model's port-i `eth_port_t` into
  switch i, for every port i (`main.c:158-160`).

So on the arch side, the arch models and the VVED bridge are all just
`eth_port_t` peers on a switch. Packets from the emulated DUT arrive over VVED,
enter via `emu_eth`, and the switch forwards them to the destination arch model
(and vice versa).

### The "dummy eth port" in libdpi

`libdpi.so`'s emu model still *creates* real `emu_eth` ports
(`ifoe_ss_model_emu.c:82-83`), but libdpi links **`libctvved_dummy.a`** — a no-op
stub (`src/libs/emu_libdpi/Makefile:36`; `ctvved_dummy.c` send does nothing,
`is_avail` returns true). So libdpi drives **no** ethernet itself; the DUT's real
ethernet is RTL MAC traffic that leaves via EPGM/VVED. Only `emu_eth_to_arch`
links the real `ctvved` (→ `libVE_TestBench_API`). This is a wiring choice, which
is why it lives here and not in the model-internals skill.

### Wiring diagram

```
 ══ libdpi.so (primary Veloce proc) ═══════════════════════════════
   ctb_cmd_input → ctctb → ctctb_cmd_handler_callback  (ifoe_emu_test_app)
        └─ runs: ifoe_push_config + sdp_ex_test + ifoe_emu_test_rdwr
   1× ifoe_ss_model_t [emu flavour]
        SDP  → emu_sdp → ctctb ─┐   CFG → real fw drv → ctctb AXI ─┐
        eth  → emu_eth → ctvved_DUMMY (no-op)                      │
                                                                   ▼
                                              Veloce emulator (RTL): SDP/AXI
                                              DUT MAC ── real eth ──► EPGM/VVED ┐
 ══ emu_eth_to_arch (VVED/EPGM host) ══════════════════════════════════════════│
   main(): N× ifoe_ss_model_t [arch flavour]                                    │
        each: ifoe_push_config + sdp_ex_test + ifoe_emu_test_rdwr (same libs)   │
        SDP/CFG run locally in-process (no cross-binary coupling)               │
   real ctvved ◄──────────────────────────────── VVED domain ──────────────────┘
        │  per IFoE port i (0..P-1):
        └─ emu_eth[i] ─┐
                       ▼
              ┌── eth_switch sw[i] ──┐   (get_dest routes on acc id)
   N arch     │  eth_port_t peers    │
   models ────┴──────────────────────┘
   (each model's port i → switch i)
   COUPLING between the two binaries is ETHERNET ONLY.
```

## Command line

This skill also covers what is specific to this variant when forming the
`velocetool-ifoe` command line. Everything generic is owned elsewhere — do not
duplicate:

- `velocetool-ifoe` itself, its full arg set, the environment-vs-structural arg
  split: `dev-knowledge-velocetool`
- The Slurm job it submits (state, cancel, `mi_veloce`): `infra-grid-atl`
- Runtime, the `--test` directory internals, where results / logs land:
  `dev-knowledge-ss-emu-environment`
- Deriving the `--rtl` er-release path from a workspace name, and er vs ner:
  `dev-workflow-ss-emu-workspace`
- Building the artifacts (`libdpi.so`, `emu_eth_to_arch`, `emu_eth_lb`):
  `build-ifoe-arch-model-ss-emu`
- ctctb / ctb_cmd routing into test code: `dev-knowledge-arch-model-transactors`
- The model internals below the interface (arch/emu/cmod flavours, config path,
  SDP/eth port implementations): `repo-ifoe-arch-model-ss-model`
- The TE variant: `dev-knowledge-ss-emu-te`

## The command line

Run `velocetool-ifoe` (see `dev-knowledge-velocetool` for its path) with the
`setup` subcommand, **from inside the workspace directory** (the `EPGM_ifoe_ss_*`
model tree):

```
velocetool-ifoe --debug setup \
  --rtl <ER_RELEASE> \
  --skip-rtl-rsync \
  --allocID MI450_P1_VEL_AINIC_AP \
  --emu-list MUGELLO \
  --constraint RHEL8 \
  --partition mi_veloce \
  --test ifoe_emu_test_app \
  --libdpi          <WORKSPACE>/ifoe-arch-model/build/src/libs/libdpi.so \
  --run-epgm-binary <WORKSPACE>/ifoe-arch-model/build/src/apps/emu_eth_to_arch
```

Omit `--gui` for a batch run (GUI usage: to be documented separately). For the
meaning/categories of the environment args (`--allocID/--emu-list/--constraint/
--partition`) and generic args (`setup`, `--debug`, `--libdpi`, `--rtl`), see
`dev-knowledge-velocetool`.

## What is specific to this variant

The **structural** args that make the run *this* variant rather than another:

- `--test ifoe_emu_test_app` — selects this variant's test directory
  `chip/runtime/vulcano_ifoe/ifoe_emu_test_app` (siblings include
  `vulcano_sdp_socapi_test`, `te_test`). This is *the* choice that makes it the
  self-contained IFoE test app. What the directory's files do is owned by
  `dev-knowledge-ss-emu-environment`.
- `--run-epgm-binary .../emu_eth_to_arch` — this variant's epgm app. Use
  `emu_eth_to_arch` for the arch-model test; `emu_eth_lb` is the loopback app.
- `--libdpi .../ifoe-arch-model/build/src/libs/libdpi.so` — the arch model built
  from *this* workspace (per `build-ifoe-arch-model-ss-emu`), not a released lib.

## `--skip-rtl-rsync` is required here

For the overlay workspace this flag is **not optional**. Without it,
velocetool's `prepare_rundir` runs `rsync … {rtl} .`, copying the **er release**
into the work area; the run would then use the *released* `chip` / `_ip` / `_env`
instead of your overlay clones (and needlessly copy a huge tree). Always pass
`--skip-rtl-rsync` so the run uses the workspace's overlay repos. (Overlay repos:
`dev-knowledge-repo-ifoe-emu-overlays`; workspace layout:
`dev-workflow-ss-emu-workspace`.)
