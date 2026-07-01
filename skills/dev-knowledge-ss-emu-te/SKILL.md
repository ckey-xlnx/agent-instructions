---
name: dev-knowledge-ss-emu-te
description: >
  The TE (Test Environment) variant of IFoE subsystem-emulation (ss-emu) — the
  ifoe_te flow. How it differs from the self-contained app: the TE build
  (ifoe-ts run.sh --build-only via lsf_bsub, producing libdpi.so + modelprovider),
  the two-phase run (start_emu.sh then start_te.sh), modelprovider as the epgm
  app, and ts-factory test packages (--tester-run). Use when working on or running
  the TE ss-emu flow (ifoe_te), building the TE artifacts, or driving TE tests, as
  opposed to the self-contained variant. For the shared runtime see
  dev-knowledge-ss-emu-environment; for the self-contained variant see
  dev-knowledge-ss-emu-selfcontained-app; for the workspace see
  dev-workflow-ss-emu-workspace.
---

# TE (Test Environment) ss-emu variant (ifoe_te)

The **TE** variant runs the same ss-emu model but drives it from the TE test
framework rather than scripting it directly via `ctb_cmd`. This is the `ifoe_te`
work area, as opposed to the self-contained `ifoe_test`
(`dev-knowledge-ss-emu-selfcontained-app`).

Shared runtime concepts (comodel hosts, libdpi, ctb_cmd, VVED/EPGM, VVED domains,
single process tree, where results/logs land) are owned by
`dev-knowledge-ss-emu-environment` — this skill covers only what is TE-specific.

> **Status:** the workspace-create and build stages below were exercised for
> real; the run stages (emu launch, test run) are transcribed from the reference
> `ifoe_emu_ci/scripts` and then **confirmed by a teammate who has this flow
> running** (values, host resolution, and the known-good `te` pin below all
> verified — note this is a teammate with hands-on experience, not the TE
> maintainers, who are another team). A couple of minor points they were unsure
> of are marked `⚠️`.
>
> The canonical sequence is checkout → build → `start_emu.sh` → `start_te.sh`
> (no separate cleanup step in normal use).

## How TE differs from the self-contained variant

| | self-contained (`ifoe_test`) | TE (`ifoe_te`) |
|---|---|---|
| epgm app | `emu_eth_to_arch` | **`modelprovider`** (an RPC provider hosting the arch model) |
| driven by | `ctb_cmd` scripts the whole test | `ctb_cmd` only boots the RPC provider; tests run in the TE framework |
| build | arch-model meson/ninja (`etx-meson`/`etx-ninja`/`emu-post-ninja`) | **`ifoe-ts` `run.sh --build-only`** via `lsf_bsub` |
| tests | the `ifoe_emu_test_app` test dir | **ts-factory packages** under `ifoe-ts/ts/…`, selected by `--tester-run` |
| launch | one velocetool call | **two phases**: `start_emu.sh` then `start_te.sh` |
| work dir | release dir keeps its name at workarea top | release renamed to `epgm_ifoe_ss` under `emul_te_build` |

## Workspace

The TE workspace is created by `ifoe_emu_ci/scripts/checkout.sh` — see
**`dev-workflow-ss-emu-workspace`** (TE section) for the full flow (release sync,
overlay swap, prebuilt libs, rename to `epgm_ifoe_ss`, sibling-repo clones, the
arch-model submodule init, and the stale-`DESIGN_NAME` caveat).

## ⚠️ Co-versioning hazard (unpinned sibling repos)

`checkout.sh` clones the sibling repos each on a **branch**, not a pinned commit
(`ifoe-arch-model@main`, `ifoe-ts@main`, `te@devel/ifoe`). These repos move
independently and there is **no manifest/lockfile** tying a known-good set
together, so a fresh checkout can pull mutually-incompatible versions and the
build will fail. (Observed: a fresh `te@devel/ifoe` was newer than the working
reference and broke the meson build with
`--native-file=/tmp/te_ws_*/native.txt: No such file or directory`.)

This is a known, long-standing limitation of the TE setup (not something these
skills can fix). If a fresh workspace fails to build this way, **pin the sibling
repos to a known-good commit set** — e.g. match the commits in a reference work
area that currently builds — rather than assuming the branch HEADs are
compatible.

**Known-good `te` pin:** `te@devel/ifoe` is currently broken. Check `te` out at
**`213bf55048d4dbc`** instead (a teammate's known-good commit that builds). So
after `checkout.sh` (or via its `-e` option), set `te` to that SHA rather than
the `devel/ifoe` HEAD.

## Build

The TE build is the `ifoe-ts` build, **not** the self-contained
`build-ifoe-arch-model-ss-emu` flow. It is run under `lsf_bsub`:

```bash
lsf_bsub -R 'select[type==RHEL8_64 && (parallel||csbatch)] rusage[mem=5000]' \
  -q regr_high -P <project> -n 4 -I \
  <ws>/ifoe-ts/scripts/run.sh --cfg=emul-ns-arch -q --build-only
```

- **Environment**: source the env script and run `bootenv` before building (the
  same cbwa init + `bootenv` used elsewhere — see
  `build-ifoe-arch-model-ss-emu` / `infra-grid-atl`). With the environment set
  up, the build submits correctly.
- **Output location** (as long as the workspace path `-w` is right, this should
  fall out): artifacts land at **`<ws>/ts_run/inst/agents/emul/libdpi.so`** and
  **`<ws>/ts_run/inst/agents/epgm/modelprovider`** — where `start_emu.sh` looks
  for them. (Mechanism, from the scripts: the inner `scripts/build.sh` does
  `cd ts_run` and sets `TE_WORKSPACE_DIR=<ws>/build`. Note the committed
  top-level `ifoe_te/build.sh` runs `run.sh` from the workspace root, which puts
  outputs at the root instead — the two committed build scripts are inconsistent,
  so go by where the artifacts actually appear.)
- ⚠️ **Do not trust `lsf_bsub -I`'s exit code** — it reported success (exit 0)
  while the inner build actually failed. Check the build log / for the artifacts.

## Run — two phases

Two CI scripts run in sequence (`IFOE_WORKSPACE=<ws>` = the `emul_te_build`
dir).

### Phase 1 — emulation (`start_emu.sh -w <ws> -p <baseport>`)
Writes `epgm_ifoe_ss/port_file`, copies the built `libdpi.so` + `modelprovider`
into `epgm_ifoe_ss/`, then launches via velocetool
(see `dev-knowledge-velocetool`):

```
velocetool-ifoe --debug setup \
  --rtl <ws>/epgm_ifoe_ss --skip-rtl-rsync \
  --test te_test \
  --emu-list MUGELLO \
  --run-epgm-binary .../modelprovider \
  --libdpi .../libdpi.so \
  --port <epgm_port> \
  --pending-behaviour automation \
  --allocID MI450_P1_VEL_AINIC_AP \
  --partition mi_high_veloce \
  --constraint RHEL8
```

`start_emu.sh` scrapes the job id from velocetool's stdout as **`Job ID: <id>`**
(velocetool itself prints `SLURM job_id is <id>` — the script reformats/parses it
to `Job ID:`; see `infra-grid-atl`).

For the `--allocID` / `--emu-list` / `--constraint` / `--partition` arg meanings
see `dev-knowledge-velocetool` (environment args); the values above are the
known-good atl+MI450 set. Note `--partition mi_high_veloce` is just a
higher-priority pick on the same emulator pool (see the partition ladder in
`infra-grid-atl`), not a TE-vs-self-contained distinction.

### Phase 2 — test (`start_te.sh -w <ws> -j <jobid> [-t <TEST_NAME>]`)
Resolves the emulation's host(s) from the job id via
`velocetool-ifoe sim_info --jobid <id> --comodel|--epgm` (a velocetool subcommand
the velocetool skill does not yet document; the `--comodel`/`--epgm` variants are
what's needed — there is a `--ctb` variant too but it is not needed here), clones
`ts-rigs`, then runs the TE tester:

```
<ws>/ifoe-ts/scripts/run.sh --cfg=emul-ns-arch \
  --veloce-host=<...> --epgm-host=<...> \
  --veloce-port=<...> --epgm-port=<...> \
  --tester-run=ifoe-ts/<TEST_NAME> \
  --ifoe-ports-num=4 --netsim-pcap -q -n
```

- **`--cfg=emul-ns-arch`** describes the test topology: emulation and the arch
  model connected via **NetSim** (the `ns`). This is the right config for a TE
  emulation run.
- **Tests are ts-factory packages** under `ifoe-ts/ts/<TEST_NAME>` (e.g. `self`,
  `basic`, `drop/nack/…`, `encapsulation/…`). **`self` is a good smoke test.**
- ⚠️ `--netsim-pcap` is optional; `--ifoe-ports-num=4` may or may not be
  relevant (unconfirmed — carried over from the reference invocation).

### Pass/fail
The tester **logs each subset of tests as passed** as it runs — that per-subset
"passed" logging is the success indicator. (For where logs land, see
`dev-knowledge-ss-emu-environment`.)

### Cleanup / teardown
There is no separate cleanup step in normal use. To tear a run down, cancel the
emulation job with `scancel <jobid>` (see `infra-grid-atl`). (`cleanup.sh -w <ws>
-j <jobid>` exists and just does that `scancel`.)

## Cross-references

- **Shared runtime**: `dev-knowledge-ss-emu-environment`
- **Self-contained variant**: `dev-knowledge-ss-emu-selfcontained-app`
- **Workspace creation**: `dev-workflow-ss-emu-workspace`
- **velocetool**: `dev-knowledge-velocetool`
- **Grid jobs / partition priority**: `infra-grid-atl`
- **modelprovider / ctctb / transactors**: `dev-knowledge-arch-model-transactors`
- **Overlay repos (chip/_ip/_env branch model)**: `dev-knowledge-repo-ifoe-emu-overlays`
- **Release trees / er vs ner / read-only snapshots**: `infra-amd-storage`
