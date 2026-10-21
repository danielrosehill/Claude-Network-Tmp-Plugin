---
name: setup
description: Set up network-tmp end to end — a capped, self-cleaning overflow directory on a home server or NAS that a RAM-limited or small-disk machine uses when /tmp is full or too small. Creates the server-side image, NFS export and hourly expiry timer, then the client automount, config and `nettmp` helper, and verifies both. Use when the user says "set up network tmp", "add another machine to network-tmp", "change the overflow size or TTL", or when `nettmp` is missing.
---

# Set up network-tmp

Two halves, each a re-runnable script in this plugin's `scripts/`:

| Script | Runs on | As | Does |
| --- | --- | --- | --- |
| `server-setup.sh` | the storage host | root | fixed-size ext4 image → loop mount → NFSv4 export to named clients → hourly cleanup timer |
| `client-setup.sh` | each machine that overflows | root | `nfs-common`, on-demand automount, `/etc/network-tmp.conf`, `/usr/local/bin/nettmp`, local fallback dir |

**Finding the scripts.** Use `${CLAUDE_PLUGIN_ROOT}/scripts/`. If that is unset
(the skill was read from a catalogue rather than an installed plugin), use a
local clone of `https://github.com/danielrosehill/Claude-Network-Tmp-Plugin`,
`git pull` first, or `gh repo clone` it into the scratchpad.

## 1. Gather the inputs

Check what already exists before asking anything:

```bash
cat /etc/network-tmp.conf 2>/dev/null; command -v nettmp && nettmp status
```

If it's already configured, this is a change (new client, new TTL), not a fresh
install. Re-run only the half that changes.

Otherwise decide, asking only for what you cannot find out:

- **Server.** Prefer an always-on Linux box with SSD/NVMe and free space over a
  NAS on spinning disks. Use its **tailnet IP** if the network has Tailscale, so
  the same address works at home and away. A LAN IP works only at home.
- **Clients.** The IP of each machine as the server will see it (the tailnet IP
  of the client if the server is addressed by tailnet IP).
- **Size** (`NETTMP_SIZE`, default `100G`). This is a hard cap. Check the
  server's free space with `df -h /` and leave it at least 20% free afterwards.
- **TTL** (`NETTMP_TTL`, default `24h`) for `scratch/`, and `NETTMP_KEEP_TTL`
  (default `7d`) for `keep/`.
- **Owner** (`NETTMP_OWNER`, default `1000:1000`). This must be the uid:gid of
  the user on the *client*, because NFS `sec=sys` maps by number. Check with
  `id` on both machines.

## 2. Server half

Copy `server-setup.sh` to the server and run it as root. Use whatever remote
execution you have: a host-control MCP tool, or `scp` plus `ssh`.

```bash
sudo NETTMP_CLIENTS="100.x.y.z" NETTMP_SIZE=100G bash server-setup.sh
```

It is idempotent. Re-running it rewrites the export list and the TTL and never
reformats an existing image. To add a client, re-run it with the full,
space-separated client list.

**Exclude the image from the server's backups.** A file-level backup of `/`
(restic, borg, rsync) skips the mount if it uses `--one-file-system`, but the
image *file* is on `/` and gets read in full every run. Add
`/var/lib/network-tmp` (or your `NETTMP_IMAGE` directory) to the backup's
exclude list. Look for the backup job (`systemctl list-timers`, crontab) and
check for this before reporting the setup as done.

Then confirm three things: `df -h /srv/nettmp` shows the image's size,
`exportfs -v` lists each client, and
`systemctl list-timers network-tmp-clean.timer` shows a next run.

## 3. Client half

On each client:

```bash
sudo NETTMP_SERVER=100.a.b.c bash client-setup.sh
```

## 4. Verify it works, not that the units are green

```bash
nettmp status
P=$(nettmp path --strict) && dd if=/dev/zero of="$P/probe" bs=1M count=50 status=none && ls -l "$P/probe" && rm "$P/probe"
NETTMP_SERVER=192.0.2.1 nettmp path        # must print the local fallback, fast
```

To prove expiry works, backdate a file's mtime past the TTL
(`touch -d '30 hours ago' "$P/old"`). Then on the server, run
`systemctl start network-tmp-clean.service` and check the file is gone.

If `nettmp path --strict` fails while the server is up, test port 2049 from the
client with `timeout 2 bash -c 'exec 3<>/dev/tcp/SERVER/2049'`. A firewall,
Tailscale ACL or missing `nfs-server` blocks it there.

## 5. Optional: stop Claude Code filling a tmpfs /tmp

Claude Code keeps its per-session scratchpads under `$TMPDIR/claude-<uid>`. On
a machine where `/tmp` is a tmpfs, that is RAM, and model weights or audio
written to a scratchpad compete with memory. Claude Code honours
`CLAUDE_CODE_TMPDIR` and creates its private `claude-<uid>/` (mode 0700) under
it. Set it to **`/var/tmp`**: that's local disk, systemd clears it after 30
days, and it's only 4 characters longer than `/tmp`. Add it to the `env` block
of `~/.claude/settings.json`, which running sessions pick up immediately.

Don't use a long path like `~/.cache/claude-tmp`. Claude Code creates sockets
under this directory, and socket paths are capped at about 104 bytes. With a
long project name, `~/.cache/claude-tmp/claude-1000/<project-slug>/…` already
reaches 102. Don't point it at the NFS mount either, because sockets don't
work over NFS.

This changes every future Claude Code session, so offer it rather than
applying it.

## Resizing

The image only grows. Stop the client mounts, then on the server:

```bash
umount /srv/nettmp
truncate -s 200G /var/lib/network-tmp/nettmp.img
e2fsck -f /var/lib/network-tmp/nettmp.img
resize2fs /var/lib/network-tmp/nettmp.img
mount /srv/nettmp
```
