---
name: dev-workflow-ss-emu-workspace
description: >
  Create an IFoE subsystem-emulation (ss-emu) workspace from a released model:
  rsync the read-only release into your work area, make it writable, swap the
  bundled chip/_ip/_env directories for clones of the overlay repos on the
  release's branch, and clone ifoe-arch-model alongside — for both the
  self-contained test-app flow and the TE flow. Use when asked to create/set up
  an ss-emu workspace, start a new emulation work area, check out a release for
  emulation, or "get a workarea ready to build the arch model". Runs at the atl
  site. After this, build with build-ifoe-arch-model-ss-emu; for the overlay-repo
  branch model see dev-knowledge-repo-ifoe-emu-overlays; for the runtime see
  dev-knowledge-ss-emu-environment.
---

# Create an ss-emu workspace

A workspace is a **writable copy of a released ss-emu model**, with IFoE's
overlay directories (`chip`, `_ip`, `_env`) replaced by git clones checked out on
the release's branch, and `ifoe-arch-model` cloned alongside to build against it.

**Site: atl.** Releases and work areas live on the atl filesystem; run this on an
atl machine (see `infra-amd-sites`).

> The committed `ifoe_emu_ci/scripts/checkout.sh` in the work areas is **stale**
> (dated Apr 2025, references an old LSC release, and for the self-contained case
> only clones `chip`). The canonical process below is derived from the live work
> areas, not that script. Treat the script as a rough reference only.

## Where releases live

Released models are under the atl "Release" trees. There are two flavours:

- **er = "export restricted"** — the full release, containing debug info (RTL,
  signal connectivity, etc.). Lives under `/proj/vulcano_dump0/Release/`. The run
  tooling references it for the RTL (e.g. `--rtl`).
- **ner = "not er"** — produced by **stripping the export-restricted debug info**
  from the er release. Lives under `/proj/vulcano_dump2_ner/Release/`. This is the
  one you sync into a work area.

Paths:

- `/proj/vulcano_dump2_ner/Release/<LSx_Design>/<DESIGN_NAME>` — the ner release
  (synced into the work area).
- `/proj/vulcano_dump0/Release/<LSx_Design>/<DESIGN_NAME>` — the er release
  (full debug / RTL).
- `/proj/vulcano_dump2_ner/Release/ifoe_tools/emul_prebuilt_libs/<LSx_Design>/<DESIGN_NAME>/`
  — prebuilt libraries (used by the TE flow).

Note the er and ner release names differ slightly: the er name has an extra `_er`
segment after the ss-number (e.g. ner `EPGM_ifoe_ss_156478_v3_…` ↔ er
`EPGM_ifoe_ss_156478_er_v3_…`).

`<DESIGN_NAME>` is the release name, e.g.
`EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z`; `<LSx_Design>` is its design dir,
e.g. `LSE_Design`. These match the overlay-repo branch names exactly (see
`dev-knowledge-repo-ifoe-emu-overlays`).

### Deriving the er (RTL) path from a workspace

A workspace directory is named after the **ner** `<DESIGN_NAME>` it was synced
from. To get the matching er (RTL) release path (e.g. for velocetool's `--rtl`):

- Prefix `/proj/vulcano_dump0/Release/<LSx>_Design/`.
- Insert `_er` after the ss-number in the name
  (`EPGM_ifoe_ss_156478_v3_…` → `EPGM_ifoe_ss_156478_er_v3_…`).

The `_er` insertion and the `dump0/Release` prefix are mechanical, **but `<LSx>`
(LSC / LSD / LSE) is NOT derivable from the workspace name** — it must be looked
up per release. So the mapping is a per-release lookup, not a pure transform.

Example (LSE):

```
workspace / ner: <...>/EPGM_ifoe_ss_156478_v3_xcb_20250822T144931Z
er (--rtl):      /proj/vulcano_dump0/Release/LSE_Design/EPGM_ifoe_ss_156478_er_v3_xcb_20250822T144931Z
```

**Releases are read-only snapshots** (`dr-xr-x---`, no write bits). That is why
the first thing you do after copying one is `chmod -R u+w`. (For *why* they are
snapshotted read-only, see `dev-workflow-ss-emu-add-release`.)

## Prerequisite

The release's branch must already exist in the overlay repos
(`pensando/ifoe-emu-chip`, `pensando/ifoe-emu-ip`, `pensando/ifoe-emu-env`). If
it does not, add it first — see `dev-workflow-ss-emu-add-release`.

## Self-contained test-app workspace (canonical)

This is the flow for self-contained test-app work (the `ifoe_test` style work
area). `$WORK` is the directory you want the work area in; `$REL` is the design
dir (e.g. `LSE_Design`); `$DESIGN` is the release name.

```bash
cd "$WORK"

# 1. Sync the read-only ner release into the work area
rsync -ra /proj/vulcano_dump2_ner/Release/$REL/$DESIGN .

# 2. Make it writable
chmod -R u+w "$DESIGN"
cd "$DESIGN"

# 3. Move the bundled overlay dirs aside and clone the overlay repos
#    on this release's branch (all three: chip, _ip, _env)
mv chip chip.old
git clone -b "$REL/$DESIGN" git@github.com:pensando/ifoe-emu-chip.git chip

mv _ip _ip.old
git clone -b "$REL/$DESIGN" git@github.com:pensando/ifoe-emu-ip.git _ip

mv _env _env.old
git clone -b "$REL/$DESIGN" git@github.com:pensando/ifoe-emu-env.git _env

# 4. Clone the arch model alongside, with its submodules
git clone git@github.com:pensando/ifoe-arch-model.git
cd ifoe-arch-model
git checkout <arch-model-branch>          # normal branch policy; e.g. main
git submodule update --init               # all three top-level submodules
```

Submodule notes (`.gitmodules` on `main` declares **three**:
`external/ifoe_ss_model`, `external/mpifoe-fw`, `external/ualoe_access_lib`):

- Run **`git submodule update --init`** (all three) — not just
  `external/mpifoe-fw/`. The build needs `ifoe_ss_model` (it defines
  `IFOE_SS_MODEL_ENCAP_TYPE_*`); initialising only mpifoe-fw leaves the build
  broken.
- Do **NOT** use `--recursive`: mpifoe-fw's nested submodules
  (`dependencies/zephyr`, `dependencies/amd-zephyr-common`) point at
  non-existent repos and abort the update. Top-level `--init` only.

Notes (from the live work area):
- The work-area directory **keeps the release name** (`$DESIGN`); it is *not*
  renamed to `epgm_ifoe_ss` (that rename is a TE thing).
- All three overlays are cloned; each leaves its bundled original as
  `chip.old` / `_ip.old` / `_env.old`.
- The overlay remotes use the hyphenated repo names (`ifoe-emu-chip`,
  `ifoe-emu-ip`, `ifoe-emu-env`). `ifoe_emu_ip` (underscores) is a valid alias
  of `ifoe-emu-ip`, so either works for `_ip`.
- The synced release's own top-level files (`*_cfg.tcl`, `P4CONFIG`,
  `cbwa_config_id`, `configuration_id`, …) stay in place — the work area *is* the
  release with the overlay dirs swapped.

You can now **build**: see `build-ifoe-arch-model-ss-emu` (bash + cbwa +
`bootenv -u emu_epgm -C $WORK/$DESIGN`, then `etx-meson` / `etx-ninja` /
`emu-post-ninja`).

### velocetool (needed to run, not to build)

The self-contained flow launches with `velocetool-ifoe`, which is **not** part of
the workspace or the release — you provide it separately. Use a per-user clone at
`/proj/vulcano_dump2_ner/<user>/smartnic-vbu` (shared across your workspaces); do
**not** use the stale `common/smartnic-vbu` copy. See `dev-knowledge-velocetool`
for how to obtain it and why, and `dev-knowledge-ss-emu-selfcontained-app` for the
command line. (The TE flow instead clones velocetool into its own workspace via
`checkout.sh` — see below.)

## TE workspace (differences)

The TE flow (`ifoe_te` style work area) does the same release-sync + overlay-swap,
but adds TE components and is fully scripted in
`ifoe_emu_ci/scripts/checkout.sh` (the TE copy of that script is current, unlike
the self-contained one). What it does differently:

- **Renames the work dir to `epgm_ifoe_ss`** after the overlay swap (so the
  release lives at `<work>/epgm_ifoe_ss/` with `chip`/`_ip`/`_env` inside).
- Clones extra repos alongside into the workspace:
  - `ifoe-arch-model` (+ submodules — see the submodule notes above; init all
    three, not `--recursive`) — branch via `-a` (default `main`)
  - `ifoe-ts` (`pensando/ifoe-ts`) — the TE test suite — branch via `-t`
  - `te` (`Xilinx-CNS/ol-test-environment`) — the test engine — branch via `-e`
    (default `devel/ifoe`). ⚠️ `devel/ifoe` is a moving branch and has broken the
    build; pin `te` to a known-good commit — see `dev-knowledge-ss-emu-te`
    (co-versioning hazard).
  - `smartnic-vbu` (`Xilinx-CNS/smartnic-vbu`) — velocetool
- **Copies prebuilt libraries** into `<DESIGN>/prebuilt/` from
  `…/Release/ifoe_tools/emul_prebuilt_libs/$REL/$DESIGN/` (self-contained has no
  `prebuilt/`).
- Driven as `checkout.sh -w <workspace> [-a <archmodel-branch>] [-t <ts-branch>]
  [-e <te-branch>] [-c]` (`-c` cleans the workspace first).

## Cross-references

- **Add a release to the overlay repos first (if missing)**: `dev-workflow-ss-emu-add-release`
- **Overlay-repo branch model**: `dev-knowledge-repo-ifoe-emu-overlays`
- **Build the model**: `build-ifoe-arch-model-ss-emu`
- **Runtime / variants**: `dev-knowledge-ss-emu-environment`, `dev-knowledge-ss-emu-selfcontained-app`, `dev-knowledge-ss-emu-te`
