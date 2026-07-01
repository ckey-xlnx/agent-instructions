---
name: dev-knowledge-ss-emu-environment
description: >
  How an IFoE subsystem-emulation (ss-emu) session is structured at runtime: the
  comodel process model, libdpi.so as the test-code entrypoint, the ctb_cmd
  driving interface, the VVED/EPGM ethernet transactor and its processes, VVED
  domains, and the single-process-tree lifecycle that kills secondary processes
  when the job ends. Use when reading, writing, or debugging how an ss-emu run
  hangs together at runtime — the primary comodel host, the EPGM/VVED ethernet
  processes, libdpi, ctb_cmd / ctb_cmd_input, stale processes after a job dies,
  VVED domains, or "where does my test code actually run". Generic across the
  self-contained test app and TE; for those see
  dev-knowledge-ss-emu-selfcontained-app and dev-knowledge-ss-emu-te. ss-emu only
  — MID emulation differs. To build the model see build-ifoe-arch-model-ss-emu;
  for the ctctb glue / transactor abstraction see
  dev-knowledge-arch-model-transactors; for grid/job mechanics see infra-grid-atl.
---

# IFoE subsystem-emulation (ss-emu) runtime environment

The generic runtime picture shared by both ways the ss-emu harness is driven
(the self-contained test app and TE). This is about how a *running* session is
structured — not how to build it (`build-ifoe-arch-model-ss-emu`) or how to
launch/query/kill the job (`infra-grid-atl`).

**Scope: ss-emu only.** This describes the subsystem-emulation platform. MID
emulation differs and is not covered here.

> Much of the detail below is dictation-sourced and cross-checked against the
> `_ip` launch scripts in one workarea; treat vendor/runtime specifics as
> subject to later validation. Where a fact lives in external Veloce/VVED
> tooling rather than our code, that is called out. Conjecture is marked.

## The comodel process model

An ss-emu session is spread across **comodel hosts** (which may be the same
physical host or different hosts):

- **Primary comodel host** — runs the main emulation process (headed or
  headless). This process:
  - loads **`libdpi.so`**, the entrypoint for all of our test code in this
    process;
  - houses the **SDP** and **AXI** transactors;
  - is driven through the **`ctb_cmd`** interface (see below).
- **Secondary comodel host** — runs the **VVED/EPGM** ethernet side (see below).

## libdpi.so — the test-code entrypoint

`libdpi.so` is the single shared object loaded into the primary emulation
process; everything we run in that process is linked into it. It is **linked,
not compiled** from one source tree — it whole-archives a set of static libs
(the model, firmware, SDP/eth emu libs, the ctb glue) plus the Veloce CTB
harness archive. The two variants build/locate it differently (self-contained
vs TE); see their skills.

## The ctb_cmd interface (how the primary process is driven)

`ctb_cmd` is the standard AMD CTB command channel (not IFoE-specific).
`ctb_cmd_input` is a **file in the run directory**. A CTB command-stream thread
in the emulation process polls it; each newline-terminated line is parsed into
`module / command / args` and dispatched through the SoCAPI requestor. This is
how a session is scripted at runtime — you write a line to `ctb_cmd_input` and
it becomes a command call inside the emulation process.

The final hop — routing a dispatched command into *our* test code inside
`libdpi.so` (the `ctctb` glue and the `ctctb_cmd_handler_callback` that the test
app implements) — is IFoE/arch-model-specific and is documented in
`dev-knowledge-arch-model-transactors`. The *set* of commands and how tests are
actually specified also differs between the variants (the self-contained app
scripts whole tests via `ctb_cmd`; TE uses `ctb_cmd` only to boot an RPC
provider and drives tests through the TE framework) — see their skills.

## The "test" directory (model / test terminology, run.veloce, ctbconf.xml)

### How the core infra names it

velocetool's `--test <name>` is passed straight through to the next-level
command, **`run_emu`**, as **`-test <name>`**, together with
**`-model vulcano_ifoe`** (from `velocetool-ifoe`: `run_emu -model vulcano_ifoe
-test <name>`). So in core-infra terms **`vulcano_ifoe` is the *model*** and the
selected directory is **the *test*** (`-test <name>`). (velocetool itself is out
of scope here — a separate concern.)

Each test is a directory under the model:
`chip/runtime/vulcano_ifoe/<test>/`, e.g. `ifoe_emu_test_app` (siblings include
`te_test`, `vulcano_sdp_socapi_test`). At run time run_emu symlinks the chosen
test directory into the session run directory as **`test_source`**
(`test_source -> …/chip/runtime/vulcano_ifoe/<test>`), which is why scripts refer
to files as `test_source/<file>`.

**This directory lives in the `chip` overlay repo**, so it is version-controlled
there alongside the rest of the chip overlay — see
`dev-knowledge-repo-ifoe-emu-overlays`. (Note `ifoe_emu_test_app` and `te_test`
themselves were added by IFoE overlay commits, e.g. IFOESW-88 / -410.)

### What is in the test directory

The files here control the run of the primary Veloce process. The load-bearing
ones:

- **`run.veloce`** — the Tcl script executed by the main Veloce process. It does
  the low-level emulation driving: downloads waveform **triggers**
  (`trigger download test_source/vul_trig.trig …`, or an absolute `.trigger`
  path, with `-onmature capture_waveform` to upload a waveform on trigger
  match), applies **forces/pokes** (`pForce emu_ifoe_ss_wrapper.… 'h1` — resets,
  test-mode straps, PHY input tie-offs), and advances time (`run 1000ns`). This
  is also where you would poke **`ctb_cmd_input`** to script the test (see the
  ctb_cmd section above). `master.veloce` defines the default Veloce step
  sequence (`emu_init` / `emu_run`); a test dir can override it.
- **`ctbconf.xml`** — CTB configuration read by the emulation harness. Sets the
  CTB `<testname>` (e.g. `cEmuSoc15HybridTest`), `<gui>`, `<debug_level>`,
  optional `<ctb_commands>`, and — importantly — the **SoCAPI hybrid-path
  definitions** that name the transactor scopes and bind them to RTL instances:
  the AXI master (`vulcano_axi_mst` → `…vul_axi_mst_xtor_inst`) and the SDP
  orig/cmpl paths (`orig_sdp0_rc`, `cmpl_sdp0_rc` → the `vul_orig_sdp0`/`cmpl`
  instances). Those scope names are exactly what `ctctb_get_sdp_orig/_cmpl` and
  `ctctb_get_axi_master` look up (see `dev-knowledge-arch-model-transactors`).
- **triggers** — `.trig` / `.tdf` / `.vtf` files (e.g. `vul_trig.trig`) that
  `run.veloce` downloads to capture waveforms at points of interest.
- **`Reg_access*.txt`, `file*.txt`, `*.now`, `*.qel`, `*.zebu`** — per-port /
  per-config register-access sequences, stimulus data, and step/hook scripts for
  other emulation flavours (the `.veloce` variants are the Veloce ones).

At run time the session run directory therefore contains `test_source` (the
symlink above), `ctbconf.xml`, and `ctb_cmd_input`, and the run is driven by
`run.veloce` reading from `test_source/`.

## VVED / EPGM — the ethernet transactor side

The emulation session's ethernet is provided by a **VVED** transactor — the
**Veloce Virtual Ethernet Device** (a Siemens/Veloce emulation tool). Our code
links against its testbench library, **`libVE_TestBench_API`** ("libVVED"), and
registers callbacks with it for sending and receiving ethernet traffic. (Note a
naming inversion in the API: VVED's "Tx callback" is our *receive* path and vice
versa — detail in `dev-knowledge-arch-model-transactors`.)

On the secondary comodel host, **two** processes must run:

1. **`epgm`** — the main EPGM application (EPGM = Ethernet Packet
   Generator/Monitor, the VVED application). It is launched **daemonised** with
   `-d` (verbatim from the Veloce VL Ethernet user guide: `-d`/`--daemon`
   enables non-interactive/headless daemon mode, driven only via the
   Controller/Testbench API). It can also be run headed, but the ss-emu launch
   path runs it daemonised. Logs to `epgm.log`.
2. **our custom application** — linked against libVVED, registers the tx/rx
   callbacks. In the self-contained variant this is `emu_eth_to_arch`; in TE it
   is `modelprovider`. Logs to `epgmapp.log`.

### Which host, and how it is found

The VVED/EPGM host is **discovered at runtime, not statically configured**. When
the EPGM DPI transactor comes up it prints, to the veloce log
(`veloce.log/tbxrun_p1.log`):

```
EPGM DPI is now running on host machine <host> in VVED Domain <n>
```

The launch logic scrapes that line for both the host and the VVED domain id, and
uses the host to know where to `ssh` the two secondary processes.

### Which custom binary — the epgm.app passthrough

The name of our custom application is **not** set in our `_ip` scripts. It
surfaces as a CTB config option in the veloce log:

```
INFO: Overriding/adding config option: epgm.app <path-to-binary>
INFO: Overriding/adding config option: epgm.app.args <args>   (optional)
```

The launch logic scrapes `epgm.app` (and optional `epgm.app.args`) from that
log; if `epgm.app` is absent it does not launch the secondary processes. The
value is injected upstream by the launcher (velocetool's `--run-epgm-binary`
becomes a `-ctb_var epgm.app=…`); that upstream wiring lives in the external
Veloce/CTB layer, not in our overlay repos.

## VVED domains

A **VVED domain** is identified by a small integer **domain id**, range 0–3 (the
`VVED Domain <n>` in the log line above; e.g. `3` in one observed run). Multiple
domains can coexist on one comodel host — the domain id is what keeps concurrent
VVED sessions on the same host separate (hence the per-domain lock files below).

### How the domain id is chosen

The id is controlled by the **`VVED_DOMAIN_ID` environment variable**, read by
the VVED/EPGM runtime (per the Veloce VL Ethernet user guide): unset → defaults
to `0`; an explicit integer → that domain is used; **`auto`** → the runtime
selects any available domain. The EPGM-DPI binary also has collision-avoidance
("…please choose another domain") consistent with bumping off a busy domain.

Two layers, and they can disagree:

1. **Our boot sets a default.** `chip/modulefile` sets `VVED_DOMAIN_ID 1` in the
   `emu_epgm` boot branch.
2. **The runtime can override it.** In an observed run every instance came up in
   domain `3` (not `1`), across two different hosts — i.e. the EPGM-DPI runtime
   resolved the actual domain itself rather than honouring the static `1`. So the
   authoritative id is the one the runtime *reports*, not the one we set going in.

Because of that, our launch logic does not trust the pre-set value — it
**scrapes the resolved id from the log line** and then **forces it onto the
processes we spawn**: it exports `VVED_DOMAIN_ID=<scraped-id>` into the remote
environment of both secondary processes (overriding the modulefile default, so
they attach to the domain EPGM actually came up in), and builds **per-domain
lock-file paths** cleaned up on teardown, e.g. `/tmp/.EPGM/.EPGMlock_<id>`,
`/tmp/.EPGM-DPI/.EPGM-DPIlock_<id>`, `/tmp/.TestBench/.TestBenchlock_<id>`.

One detail left unconfirmed: whether the override to `3` happened because `auto`
was set somewhere, or because the binary auto-bumped off a busy domain `1`.
Either way the resolved id is runtime-determined and only known from the log.

## Single process tree (lifecycle / avoiding stale processes)

The job server (Slurm) kills the **primary** process when the job ends. That
would otherwise leave the **secondary** EPGM processes on the other comodel host
running as stale processes. To prevent this, the secondary processes are made
part of the primary's **process tree**:

- A hook inside the main run process `fork()`s; the child (`launch_epgm_procs`)
  waits for the EPGM host/domain to be reported, then launches the secondary
  processes by `exec`ing **`ssh -tt <epgm_host> …`**.
- Because the child `exec`s ssh (replacing itself), the ssh client is a direct
  descendant of the main `run_emu` process. The remote shell runs with
  `huponexit` and an EXIT trap that kills the remote app and removes its
  per-domain lock files.
- When the main process dies, the local ssh dies, the remote login shell gets
  SIGHUP / exits, and the trap tears down the remote EPGM processes — so the
  whole session dies as one tree.

**Where the code lives:** this single-process-tree machinery is **IFoE's own
custom code**, not stock Veloce behaviour. It is maintained across IFoE's three
model-workarea overlay repos — **`_chip`**, **`_ip`**, and **`_env`** — which
overlay content into a released subsystem-emulation model. The hook seen here is
`_ip/emu_global/chip/lib/emuEnvVLRunChip.pm` (`sub sim` forks;
`launch_epgm_procs` / `exec_epgm_proc` do the scraping and ssh launch),
introduced under Jira IFOESW-410, with related changes in `_env`. Dedicated
repo-reference skills for `_chip` / `_ip` / `_env` are still to be written; add
them later and cross-link here.

(Note: in the one workarea inspected, the hook was found in `_ip` with minor
`_env` changes; a `_chip` *overlay* repo was not observed there. The split
across the three repos is per the maintainer; confirm exact placement when the
repo skills are written.)

**Debugging implication:** if you find stale EPGM processes on the secondary
host, the process-tree teardown failed — check the ssh hook and the remote
EXIT-trap path rather than assuming the job itself is still alive.

## Where to go next

- **Self-contained test app variant** (ifoe_test): `dev-knowledge-ss-emu-selfcontained-app`
- **TE variant** (ifoe_te): `dev-knowledge-ss-emu-te`
- **ctctb glue + how the model abstracts over SDP/AXI/ethernet transactors**: `dev-knowledge-arch-model-transactors`
- **Building the model**: `build-ifoe-arch-model-ss-emu`
- **Launching / querying / killing the job**: `infra-grid-atl`
