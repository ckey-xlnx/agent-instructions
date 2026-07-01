---
name: infra-amd-storage
description: >
  Reference for how storage is laid out across AMD's IFoE dev environment: the
  storage tiers and their properties — machine-local (/scratch, home quota) vs
  shared NFS (/proj/...), tool installs (/tool autofs), and release trees.
  Explains the er vs ner (export-restricted vs stripped) distinction,
  read-only-snapshot vs writable-workarea, and the messy cross-site sync of
  /proj. Use when deciding where to put build output, logs, or checkouts; when a
  path is "not found" or too small/full; when you see /proj, /scratch, /tool,
  er/ner, or dump0/dump2_ner; or when reasoning about whether a path is local,
  shared, or site-specific. For sites and how to get a machine see
  infra-amd-sites; for specific well-known machines see infra-amd-machines.
---

# AMD IFoE storage & path layout

How storage is organised in the IFoE dev environment, and the properties that
decide where things should live. It is genuinely messy; this documents the
model and the *verified* facts, and is meant to grow as specifics are confirmed.

**This is a companion to `infra-amd-sites` (what a site is / how to get a
machine) and `infra-amd-machines` (which machine for what). Storage locality
only makes sense together with those.**

## The storage tiers at a glance

| Location | Kind | Scope | Notes |
|----------|------|-------|-------|
| `/scratch/...` | local disk (ext4) | **machine-local** | fast, not shared; assume not backed up; lost if you move machines |
| `$HOME` (`/home/$USER`) | home | per-user | **quota-constrained** — do NOT put large build output here |
| `/proj/...` | NFS via autofs | shared, **site-local** (messy sync — see below) | the main shared area; where big/shared stuff goes |
| `/tool`, `/tools` | autofs | per-machine (OS-dependent) | tool installs; see "Tool installs" |

Rule of thumb: **big or shared → `/proj`; fast/throwaway → `/scratch` (machine-local); never rely on `$HOME` for size.**

## Machine-local storage

- **`/scratch`** is a local disk on the machine (verified: local ext4 volume,
  not NFS). It is therefore **machine-specific** — not visible from other hosts,
  not synced anywhere, and effectively gone if your machine changes (relevant on
  etx-allocated sites where the host is transient; see `infra-amd-sites`).
  Assume it is not backed up. Good for fast scratch and throwaway build trees;
  bad for anything you need to keep or access elsewhere.
- **`$HOME`** is quota-constrained. Large outputs (build packages, diags
  binaries, caches) must not go here. (The `env-workflow-vscode-server-cleanup`
  skill exists precisely because home quota fills up.)

## Shared storage: `/proj/...`

`/proj` is an **autofs** mount that automounts an **NFS export per subtree** —
e.g. `/proj/smartnic` and `/proj/verif_release_ro` are separate NFS filers,
mounted on demand. This is the main shared/collaborative storage.

**Site-locality is real but messy.** As far as is known, `/proj` mounts are
**site-local** — the same `/proj/...` path at a different site is a different
filesystem (see `infra-amd-sites`). But the same logical content is often kept
in sync across sites by **ad-hoc machinery**, and the arrangement varies
case-by-case:
- sometimes a **master** copy with **read-only slaves** at other sites;
- sometimes **all read-write** with **one-way sync** in some direction;
- sometimes ad-hoc `rsync` jobs.

So: do not assume a `/proj` path is writable, current, or even present at a
given site. **Document specific `/proj` trees' site/sync behaviour here only
when established for a fact** (table below), and treat everything else as
unverified.

### Releases vs work areas (two content patterns)

Two kinds of content live on these volumes, distinguished by lifecycle rather
than by location — **they are intermixed within the same mount points** (e.g.
`/proj/vulcano_dump2_ner/` holds both the ner `Release/` trees and per-user work
areas). Recognise which you are looking at:

**Release** — a published, immutable artifact (an ss-emu release, an installed
SimNow build, etc.).
- The directory is writable *while being produced*, but its contents are made
  **read-only once written** — e.g. ss-emu release trees end up `dr-xr-x---`
  (no write bits); an installed SimNow release is `chmod -R ugo-w+r` after
  unpack so processes can't drop tempfiles into it.
- Treat a release as a stable, immutable base. Don't work in it directly.

**Work area** — a per-user working copy.
- Named for the user (e.g. `/proj/vulcano_dump2_ner/<user>/...`); the user does
  whatever they like inside it.
- Typically created by copying a release and making the copy writable
  (`chmod -R u+w`), then modifying it (for ss-emu: swapping in overlay clones —
  see `dev-workflow-ss-emu-workspace`).

General pattern: **releases are immutable and shared; you work in your own
writable copy.**

### er vs ner (export-restricted vs stripped)

A recurring distinction in release trees and some path names:

- **er = "export restricted"** — the full release, **including** export-restricted
  debug info (RTL, signal connectivity, etc.). Lives under
  `/proj/vulcano_dump0/Release/`. Run tooling references it for RTL (e.g.
  velocetool `--rtl`).
- **ner = "not er"** — the same release with the export-restricted debug info
  **stripped out**. Lives under `/proj/vulcano_dump2_ner/Release/`. This is the
  one you sync into a work area.

The two release names differ only by an `_er` segment after the ss-number
(ner `EPGM_ifoe_ss_156478_v3_…` ↔ er `EPGM_ifoe_ss_156478_er_v3_…`). See
`dev-workflow-ss-emu-workspace` for deriving one from the other.

### Known `/proj` trees (verified facts only — grow this)

Each tree below lists its purpose, then a **per-site** breakdown: the access
mode at that site (**ro**/**rw**) and, if the content is synced, **where it is
synced from**. A tree replicated to several sites appears with one row per site.
Only record a site row when it is known for a fact; leave the rest out rather
than guess. `common` = deliberately cross-site / not tied to one site.

**`/proj/smartnic/xcb/ifoe/...`** — IFoE content within the `smartnic/xcb`
project storage: SimNow installs (`.../ifoe/simnow/`), diags caches/artifacts
(`.../ifoe/diags/`), etc. **Within `/proj/smartnic/xcb/`, all IFoE-related
content lives under the `ifoe/` subdir** (other `smartnic/xcb` subdirs are
non-IFoE and out of scope here).
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| unverified | unknown | unknown | NFS export `xcbswsvm2-lif2:/smartnic` observed on one host; the `xcb` token suggests xcb-origin but single-site vs replicated is unconfirmed |

**`/proj/verif_release_ro/`** — cbwa init script + verif release content (name
marks it read-only).
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| unverified | unknown | unknown | NFS export `xcbitvs1-lif3:/cache10001/verif_release` observed |

**`/proj/vulcano_dump0/Release/`** — **er** release trees (see er/ner below);
read-only snapshots.
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| atl | ro | unknown | per ss-emu skills |

**`/proj/vulcano_dump2_ner/Release/`** — **ner** release trees; read-only
snapshots, synced into workareas.
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| atl | ro | unknown | per ss-emu skills |

**`/proj/vulcano_dump2_ner/`** (non-Release) — big-storage working area:
packaging output, workareas, common tool copies (e.g. `common/smartnic-vbu`).
**Backed up** (unlike `/scratch`), so safe for work you need to keep.
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| atl | rw | n/a (working area) | per ss-emu skills; large writable, backed-up area |

Personal workspaces live under a per-user directory here:
`/proj/vulcano_dump2_ner/<user>/...` (the author's is
`/proj/vulcano_dump2_ner/ckey/...` <!-- personal -->). Per-activity subdirs
below that are a personal convention — see the user's env-layout skill.

**`/proj/mi450_runs/`** — diag_tng builds.
| Site | ro/rw | Synced from | Notes |
|------|-------|-------------|-------|
| dfc | rw | n/a | per build-diag-tng |

(`/proj/emu/slurm/wrapper/bin/` — the Slurm `squeue`/`scancel` wrappers — is a
tool location rather than a storage tree; see `infra-grid-atl` / `infra-amd-machines`.)

(Site rows are recorded from what skills state or what was observed on a single
host; "unverified" means single-site-vs-replicated is not confirmed. Fill in
ro/rw and sync-source per site as each is established for a fact.)

## Tool installs: `/tool` and `/tools`

`/tool` and `/tools` are **autofs** automounts holding tool installs. The
behaviour below is established only for the **Pandora** toolset under them
(`/tool/pandora`, `/tool/pandora64`, `/tools/pandora`, `/tools/pandora64`); do
not assume it generalises to other things under `/tool`/`/tools`.

- **Pandora path forms are interchangeable.** On a tested host, `/tool/pandora`,
  `/tool/pandora64`, `/tools/pandora`, `/tools/pandora64` all resolve to the
  **same underlying directory** (verified by inode). Skills use these forms
  inconsistently for this reason; they are believed equivalent.
- **Pandora tools vary per-machine (OS build), not per-site.** What is available
  and which OS build (RHEL7, RHEL8, …) depends on the machine you are on — a
  Pandora tool path that works on one host may be absent or a different build on
  another. Match the machine/OS to the tool (see `infra-amd-machines`).
- Pandora tools live under `.package/<name>-<version>/bin/`, e.g.
  `/tool/pandora64/.package/python-3.14.0/bin/python3`.

Whether the wider `/tool`/`/tools` trees are site-independent or OS-dependent is
**not established** — document other tools' behaviour here as it is confirmed.

## See also

- `infra-amd-sites` — what a site is, how to get a machine, VDI vs etx.
- `infra-amd-machines` — specific well-known machines and what each is for.
- `dev-workflow-ss-emu-workspace` — release-sync + workarea creation (er/ner in practice).
- `admin-workflow-simnow-install-release` — the read-only-after-unpack install convention.
