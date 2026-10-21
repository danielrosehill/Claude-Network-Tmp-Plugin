---
name: cleanup-network-tmp
description: Report and clean the network-tmp overflow — show how full the share and the local /tmp are, what this machine has stored and how old it is, and delete it now instead of waiting for the automatic expiry. Use when the user says "clean up network temp", "clear the overflow", "how full is network tmp", "free space on nettmp", or when the share is filling up.
---

# Clean up network-tmp

Automatic expiry already runs hourly on the server: anything in `scratch/`
untouched for `NETTMP_TTL` is deleted, and the same goes for `keep/` after
`NETTMP_KEEP_TTL`. This skill is the manual, immediate version.

## 1. Report

```bash
nettmp status          # human
nettmp status --json   # for parsing
```

Report the share's used/free space, this host's usage and oldest file, and the
local `/tmp` percentage. If `/tmp` is a tmpfs (`findmnt /tmp`), say that it
lives in RAM.

To see what's taking the space before deleting:

```bash
du -sh "$(nettmp path)"/* 2>/dev/null | sort -h | tail -15
```

## 2. Clean

Always dry-run first and show the user the result:

```bash
nettmp clean --dry-run                    # files older than the TTL
nettmp clean --dry-run --older-than 6h
nettmp clean --dry-run --all              # everything this host stored
```

Then run the same command without `--dry-run`. With no option, a dry run of
the default TTL needs no confirmation. Ask before running `--all`, or an
`--older-than` shorter than about 1h, if another session might be using the
files.

`nettmp clean` only touches **this host's** directories: `scratch/<host>-<user>`
on the share and the local fallback. It never touches `keep/` or other
machines' files. To clear `keep/` or the whole share, do it on the server
deliberately:

```bash
# on the server
systemctl start network-tmp-clean.service          # apply the TTL rules now
find /srv/nettmp/scratch -mindepth 1 -delete        # wipe all scratch, every host
```

## 3. If local /tmp is the problem

If `/tmp` is full but the share is fine, the culprit is usually large files
that should have gone to network-tmp. Find them with:

```bash
du -xsh /tmp/* 2>/dev/null | sort -h | tail -10
```

On a machine running Claude Code, the usual cause is old session scratchpads
under `/tmp/claude-<uid>/`, which hold model files and recordings. List them
and ask before deleting any session directory that might still be running. A
directory modified within the last hour may belong to a live session.
