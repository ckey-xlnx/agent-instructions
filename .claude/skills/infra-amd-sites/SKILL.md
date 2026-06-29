---
name: infra-amd-sites
description: >
  Reference for AMD sites (offices) and their site codes, used to interpret
  machine names and NFS paths. Use when you see a site code or site-prefixed
  path (e.g. "atl:/proj/...", "the ATL filesystem", "xcb-ckey-l1"), when asked
  which site a host or path belongs to, when a command must run "on the ATL
  filesystem" or at a particular site, or when a local filesystem path is not
  found because it lives at another site. Covers what a site is, the shared
  properties of sites, and the list of known site codes (atl, xcb, xsj, dfc,
  sac, cac).
---

# AMD Sites Reference

## What a site is

A **site** is (roughly) an AMD office with its own **local filesystems**. Each
site has a short code, generally 3 characters.

Shared properties of sites:

- **Local filesystems.** Each site has its own local storage. A path like
  `/proj/...` is local to a particular site; the same path at another site is a
  different filesystem (or does not exist there).
- **Machine-name prefix.** Machines at a site are named with the site code as a
  prefix, e.g. `xcb-ckey-l1` is an `xcb` machine.
- **Site-prefixed NFS paths.** Documentation sometimes prefixes a site code to
  an NFS path to disambiguate which site's filesystem is meant, e.g.
  `atl:/proj/...` means `/proj/...` on the **atl** site.
- **Cross-site DNS.** DNS resolves across all sites — you can name a host at any
  site from any site.

**Operational implication:** to act on a site's local filesystem (e.g. "run this
on the ATL filesystem"), you must run the command on a machine *at that site*.
Referencing the bare path on a machine at a different site will not find it, even
though DNS for the remote host resolves. If a `/proj/...` path is "not found"
locally, suspect it lives at another site.

## Known site codes

| Code  | Office / location                      |
|-------|----------------------------------------|
| `atl` | Atlanta                                |
| `xcb` | Xilinx Cambridge                       |
| `xsj` | Xilinx San Jose                        |
| `cac` | CACO Cadence Colo                      |
| `dfc` | Dallas/Fort Worth Data Center (DFDC)   |
| `sac` | Sacramento Data Center (SADC)          |
