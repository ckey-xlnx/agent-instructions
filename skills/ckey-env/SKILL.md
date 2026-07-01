---
name: ckey-env
description: >
  Christopher Key's personal working environment: where ckey keeps things —
  version-control checkouts, build/source areas, ss-emu workareas, scratch,
  packaging output — and which machines ckey uses at each site. Use when a task
  needs ckey's actual paths or machines (e.g. "where is my mpifoe-fw checkout",
  "my ss-emu workarea", "my ATL box"), or to resolve the `<!-- personal -->`
  path examples in other skills to their real locations. Personal to ckey;
  teammates have their own layout. For the generic storage/site/machine model
  this builds on, see infra-amd-storage, infra-amd-sites, infra-amd-machines.
---

# ckey's environment & layout

Where **ckey** personally keeps things and which machines ckey uses. This is the
concrete, personal counterpart to the generic `infra-amd-{storage,sites,machines}`
skills — read those for *what the tiers/sites/machines mean*; read this for
*ckey's actual choices within them*.

> **This churns.** Layout is migrating (see below) and the ATL machine changes
> on a ~monthly timescale. Facts that go stale are **timestamped** — treat a
> dated line as "true as of then", and re-verify before relying on it. Undated
> lines are stable conventions.

## Version-control checkouts

- **`~/hg`** (`/home/ckey/hg`) — the general home for VC checkouts
  (`ifoe-arch-model`, `mpifoe-fw`, `mpifoe-fw-2`, and many more). On **NFS home**
  (xcb), so it counts against home quota.
  - **Problem / migration in progress:** builds here have grown large
    (~21G as of 2026-07-01) and this is on NFS home. Source/build work is
    **moving to `/proj/smartnic/xcb/ckey/src`** (below). Expect `~/hg` to shrink
    over time as things move out.
  - Skills that reference a personal `~/hg/...` checkout (e.g.
    `dev-workflow-simnow-launch`'s `$MPIFOE_FW=/home/ckey/hg/mpifoe-fw`) point
    here — but see the migration note; the canonical location may move.

## Build / source areas

- **`/proj/smartnic/xcb/ckey/src`** — xcb project-storage build/source area, the
  destination `~/hg` content is migrating to (bigger, off home quota; xcb NFS).
  Holds sibling source checkouts, as of 2026-07-01:
  `amd-tee3.0`, `asp-fmc`, `diag_tng`, `mpio`, `simnow`, `sw-security-tools`,
  `toolchain`. This is the `$SRC` in `build-asp` (where amd-tee3.0 +
  sw-security-tools + toolchain sit as siblings).
- **diag_tng build caches** (per `build-diag-tng`): Conan/ccache homes are
  per-site — `/home/$USER/...` on XCE, `/proj/mi450_runs/users/ckey/...` on DFC.

## ss-emu / emulation workareas (ATL)

Under **`/proj/vulcano_dump2_ner/ckey/`** (ATL big-storage, backed up). Verified
present 2026-07-01:

- **`ifoe_test`** — self-contained ss-emu test-app workarea (holds `.trigger` /
  `.do` test files). Matches the `dev-workflow-ss-emu-*` self-contained flow.
- **`ifoe_te`** — TE (ifoe_te) workarea (`build.sh`, `runall.sh`,
  `emul_te_build`, `ifoe_emu_ci`). Matches `dev-knowledge-ss-emu-te`.
- Other dirs seen alongside (2026-07-01): `ifoe-arch-model`, `ss_emu`,
  `simnow`, `simnow-runs`, `diag_tng`, `diags_conan`, `smartnic-vbu`,
  `bug_dumps`. This `ckey/` root is the personal working root referenced
  generically as `/proj/vulcano_dump2_ner/<user>/...` in `infra-amd-storage`.
- Personal one-off scripts live in these workareas (`ifoe_test/goall.sh`,
  `ifoe_te/runall.sh`) — convenience wrappers, not shared tooling.

## Scratch

- **`/scratch/ckey/...`** — machine-local scratch (e.g. `/scratch/ckey/simnow`
  used by `dev-workflow-simnow-launch`). Local disk, **per-machine** and not
  backed up: it only exists on the host where it was created, so the exact path
  is present only on whatever machine ckey last used it on.

## Packaging / output

- **`/proj/vulcano_dump2_ner/ckey/simnow/packaging`** — SimNow packaging output
  (`$OUT` in `build-simnow`). ATL big-storage; also `simnow-runs` alongside.

## Machines

| Role | Machine | Stability | Notes |
|------|---------|-----------|-------|
| xcb VDI | `xcbckey42x` | **stable** (years) | persistent per-user VDI at xcb; the day-to-day box |
| xsj VDI | `xsjckey41x` | may expire | persistent VDI at xsj |
| ATL etx | `atletx8-reg111` (ssh `ckey@atletx8-reg111`) | **churns ~monthly** — as of 2026-07-01 | etx-allocated, transient; **1-month timeout if not connected via etx**, then a new host. Re-confirm before use. See `infra-amd-sites` (etx). |

## SSH keys & agent

ckey uses an **ssh-agent**, with **separate keys per site and per purpose**
(e.g. distinct keys for different GitHub accounts). Keys are added to the agent
but **time out after 24h** — so a first sign of trouble after a day is
authentication failure.

- **If ssh suddenly fails** (`Permission denied (publickey,...)`, sometimes
  `Too many authentication failures`), the likely cause is **expired agent
  keys** — re-add them to the agent. (This is the auth-failure symptom noted in
  `infra-amd-sites` under etx caveats, and it bit a git push mid-session.)
- **Agent forwarding does NOT help here.** `~/.ssh/config` refers to keys **by
  path** for specific purposes (e.g. per-GitHub-account keys), and those key
  files are **site-local** — forwarding an agent from another site won't satisfy
  a config that expects a specific local key path. Keys must be present and
  loaded on the machine actually making the connection.

## See also

- `infra-amd-storage` — storage tiers, /proj sync, er/ner, release-vs-workarea.
- `infra-amd-sites` — sites, enclaves, VDI vs etx.
- `infra-amd-machines` — which machine class for which task.
