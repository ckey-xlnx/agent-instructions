---
name: admin-workflow-simnow-install-release
description: >
  Download, unpack, and permission-fix a SimNow release from Artifactory.
  Use when asked to install a SimNow release, fetch/download a new SimNow
  build, set up a release under /proj/smartnic/xcb/ifoe/simnow, or update
  to a newer SimNow version.
---

# SimNow Release Installation

## Step 1: Find the release

Releases are announced on the SimNow Test Releases Confluence page:
https://amd.atlassian.net/wiki/spaces/PFO/pages/1708294538/SimNow+Test+Releases

**Note**: This URL is subject to change. If it doesn't resolve, ask the team for the current location.

Each release entry lists one or more zip file URLs on Artifactory (`atlartifactory.amd.com`). Expect:
- `rhel8` platform
- `ifoe` project

If multiple options are listed for a release and it is not clear which to fetch, **ask the user** before proceeding.

## Step 2: Decide what to download

A release may include some or all of:
- **Release build** — standard optimised build (e.g. `...-mi450_ifoe-DATE-...-release-...zip` or no build-type suffix)
- **Debug build** — debug symbols, less optimised (e.g. `...-debug-...zip`)
- **Optimised debug build** — may also be present

Download whatever is listed. However:
- If **debug** is absent: note it (informational)
- If **release** is absent: **flag this loudly** — the release build is the primary deliverable

## Step 3: Download

Install location:

```
/proj/smartnic/xcb/ifoe/simnow/
```

Download each zip to this directory:

```bash
cd /proj/smartnic/xcb/ifoe/simnow
wget --user ${USER} --ask-pass <url>
```

For non-interactive / automated use, two alternatives:
1. **`.netrc` file** — add to `~/.netrc`:
   ```
   machine atlartifactory.amd.com login <username> password <api-token>
   ```
   Then use `wget --netrc-file ~/.netrc <url>`. Ensure `chmod 600 ~/.netrc`.
2. **curl with API token** — generate a token in the Artifactory UI and use:
   ```bash
   curl -H "X-JFrog-Art-Api: <token>" -O <url>
   ```
   Prefer an API token over your real password in either case.

## Step 4: Unpack

The zip files contain a path-prefixed directory structure. Unzip with cwd as the install root so they unpack correctly:

```bash
cd /proj/smartnic/xcb/ifoe/simnow
unzip <zipfile>
```

## Step 5: Fix permissions

After unpacking, fix permissions on the extracted directory:

```bash
chmod -R ugo-w+r <extracted-directory>
```

This does two things:
- Adds read permission for all (some files may be unreadable as shipped)
- Removes write permission for all (prevents tempfiles being created inside the installation, which would break it)

The extracted directory name comes from the zip — it will match the zip filename without the `.zip` extension.
