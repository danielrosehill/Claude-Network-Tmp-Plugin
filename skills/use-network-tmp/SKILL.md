---
name: use-network-tmp
description: Route large temporary files — model weights, checkpoints, ONNX exports, audio and video, datasets, archives, build outputs — to the network-tmp overflow instead of /tmp or the session scratchpad, and fall back to local disk when the server is unreachable. Use before downloading or generating anything over about 50 MB on a machine with network-tmp configured (`/etc/network-tmp.conf` exists), when /tmp is a tmpfs or nearly full, or when a write fails with "No space left on device".
---

# Use network-tmp

`nettmp` prints a writable directory: the network overflow when the server is
reachable, otherwise a local-disk fallback (`/var/tmp/network-tmp/<user>`).
Either way, everything in it is **deleted automatically** after the TTL
(default 24 h since last write).

## When to use it

Check `[ -r /etc/network-tmp.conf ]`. If that file is absent, this machine isn't
set up, so offer the `setup` skill instead.

| Situation | Where it goes |
| --- | --- |
| Anything expected to be over `NETTMP_LARGE_MB` (default 50 MB): weights, `.onnx`, `.safetensors`, audio/video, datasets, tarballs, model caches | `$(nettmp path)` |
| `/tmp` over about 80% full (`nettmp status` shows it) | `$(nettmp path)` for anything new |
| Small scratch files, scripts, notes | the session scratchpad as usual |
| Sockets, lock files, SQLite databases, anything opened with `mmap` or locked | **never** network-tmp, use local disk |
| Anything that must survive past the TTL | the project or real storage, not here |

Check where `/tmp` actually lives with `findmnt /tmp`. If it is `tmpfs`,
writing a large file there consumes RAM, not disk.

## How

```bash
D=$(nettmp path)                 # or: nettmp path my-job  → a named subdir
D=$(nettmp path --strict)        # fail rather than fall back to local disk
F=$(nettmp mktemp)               # unique file;  nettmp mktemp -d  for a dir
```

For tools that take a temp or cache dir, point them at it for that command
only, rather than exporting it globally:

```bash
TMPDIR=$(nettmp path) some-tool ...
HF_HOME=$(nettmp path hf) python train.py
pip download -d "$(nettmp path wheels)" ...
```

Say in your reply which directory you used and that it expires.

**Keeping something longer.** `nettmp keep <path>` moves it into `keep/`, which
expires after `NETTMP_KEEP_TTL` (default 7 d). It is still temporary: copy real
results into the project.

## Cautions

- The mount is `soft`, so if the server drops mid-write, the write fails with
  EIO after about 15 s instead of hanging. Treat a failure there as a network
  problem: rerun `nettmp path` (it will fall back) and retry.
- Throughput is network throughput. It's fine for staging files, but for heavy
  random I/O (training data loaders, compilers), copy the working set to local
  disk first.
- Away from home over a slow link, prefer `nettmp path` (it falls back) over
  `--strict`.
