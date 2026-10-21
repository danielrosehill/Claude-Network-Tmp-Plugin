# network-tmp

A Claude Code plugin, plus three shell scripts, that give a machine an
**overflow temp directory on a home server or NAS**. The directory has a hard
size cap, is mounted only when used, falls back to local disk when the server
is out of reach, and **wipes itself every 24 hours**.

It's built for laptops where `/tmp` is a small **tmpfs**. That's the default on
recent Ubuntu, and it means `/tmp` lives in RAM. Each large file written there
(model weights, audio, datasets) takes memory away from everything else. It
also helps any machine whose disk is small.

## Skills

| Skill | Use it to |
| --- | --- |
| `setup` | Build the server half and the client half, and verify both. Re-run it to add a machine or change the size or TTL. |
| `use-network-tmp` | Tell Claude to send large temporary files to the overflow instead of `/tmp` or its scratchpad. |
| `cleanup-network-tmp` | Report usage and delete this machine's overflow files now (dry run first). |

## Pieces

```
server  /var/lib/network-tmp/nettmp.img   fixed-size ext4 image (the cap)
        /srv/nettmp/scratch/<host>-<user>  expires 24h after last write
        /srv/nettmp/keep/<host>-<user>     expires 7d after last write
        /etc/exports.d/network-tmp.exports NFSv4, named clients only
        network-tmp-clean.timer            hourly systemd-tmpfiles --clean

client  /mnt/nettmp                        systemd automount, soft, 15 s timeouts
        /var/tmp/network-tmp/<user>        local fallback, same TTL
        /etc/network-tmp.conf              config
        /usr/local/bin/nettmp              helper
```

```bash
nettmp path [--strict] [SUB]    # writable dir: network, else local fallback
nettmp mktemp [-d]              # unique file/dir inside it
nettmp keep PATH...             # move to keep/ (7 d)
nettmp status [--json]
nettmp clean [--dry-run] [--all | --older-than 6h]
```

## Quick start without Claude

```bash
# on the server (root)
NETTMP_CLIENTS="100.64.0.10" NETTMP_SIZE=100G bash scripts/server-setup.sh
# on each client (root)
NETTMP_SERVER=100.64.0.1 bash scripts/client-setup.sh
nettmp status
```

Both scripts are idempotent. See `skills/setup/SKILL.md` for every variable and
for how to resize.

## Design decisions

- **The cap is a loop-mounted image file, not an LV or a quota.** It works on
  any ext4 or btrfs host, including one whose volume group has no free space,
  and the overflow can never fill the host's root disk. `-m 0` because it's
  scratch.
- **The server cleans up, not the client.** Files expire on schedule even when
  the laptop is off or travelling. An hourly timer replaces the stock daily
  `systemd-tmpfiles-clean.timer`; with the daily one, a file could live up to
  TTL + 24 h.
- **Age is judged on mtime only** (`m:24h` in tmpfiles.d). By default tmpfiles
  also counts atime and ctime. ctime is bumped by `mv`, `chmod` and
  `touch -d`, so files moved into `keep/` or backdated for testing wouldn't
  expire on schedule. Found while testing on 2026-10-21: a file backdated 30 h
  survived the cleanup run until the prefix was added. `nettmp keep` touches
  files before moving them because `mv` keeps the old mtime.
- **Mounted on demand and `soft`, never always-on and `hard`.** An always-on
  hard NFS mount hangs every shell that touches it once the machine leaves the
  network. `nettmp` also runs a 2 s TCP probe of port 2049 first, so with the
  server gone it falls back in milliseconds instead of waiting out the 15 s
  mount timeout.
- **Address the server by tailnet IP** where Tailscale exists, so the same
  mount works at home and away. Expect it to be slow over a long-haul link,
  which is why there's a fallback.
- **NFS `sec=sys`, mapped by uid.** The tree is owned by `NETTMP_OWNER`
  (default `1000:1000`), which must match the client user's uid. This is
  simple and fine for a single-user home setup. It is not a multi-tenant
  design.
- **Not for sockets, locks, SQLite or mmap.** That's NFS's limit, not the
  plugin's. Claude Code's own temp dir (`CLAUDE_CODE_TMPDIR`) holds sockets, so
  it should be moved to *local disk*, never onto this mount (see the `setup`
  skill, step 5).

## Tested

2026-10-21. Server Ubuntu 26.04 (nfs-kernel-server 2.8.5), client Ubuntu
26.04, over Tailscale. Confirmed: 200 MB write over NFSv4.2, server-side
expiry of a file backdated past the TTL with the fresh file kept, `keep`
resetting the clock, fallback to local disk with an unreachable server, and
`--strict` exiting 1.

Not tested: a NAS (Synology etc.) as the server (the scripts assume a Debian or
Ubuntu host with systemd), and behaviour when the server drops mid-write.

## License

MIT
