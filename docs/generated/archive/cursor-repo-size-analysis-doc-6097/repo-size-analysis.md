# Why is this repo ~1 GB to clone, when the original Python project is ~1 MB?

> Short answer: **this repo ships every prebuilt binary (Linux, macOS, Windows, OpenWRT, Raspberry Pi, and 5 Android APKs) for every release, committed directly into git history.** The Python project ships only source code. Compiled artifacts are big, and once they are committed, every past version stays inside `.git/objects` forever — even after you delete them from the working tree.

This document explains, with measurements taken from the current clone, exactly where the ~1 GB comes from and what would have to change to make a clone small.

---

## 1. The measurements

Run inside a fresh clone:

```bash
du -sh .                  # 1.2 GB total
du -sh .git               # 990 MB  ← the git history
du -sh .git/objects/pack  # 990 MB  (it's all one packfile)
du -sh releases/          # 157 MB  ← the prebuilt binaries on disk right now
du -sh src/ android/ docs/ assets/ tunnel-node/   # ~1.5 MB combined
```

So:

| Bucket | Size | % of clone |
|---|---:|---:|
| `.git/` (history) | **990 MB** | **~83%** |
| `releases/` (current binaries) | 157 MB | ~13% |
| Source + docs + assets + Cargo.lock + everything else | ~1.5 MB | <0.2% |

The actual Rust source code is **smaller** than the original Python project. The 1 GB is not source; it is binary artifacts and their git history.

## 2. Where the 990 MB inside `.git` comes from

`.git` stores every blob ever committed. Summing all blob sizes ever added to history:

```
Total blob bytes:        2,035,924,910  (~1,941 MB)
└─ blobs under releases/: 2,012,194,471  (~1,919 MB)
└─ everything else:           23,730,439  (~23 MB, dominated by Cargo.lock revisions)
```

So **99% of every byte ever committed to this repo is files under `releases/`.**

Git's packfile then deflates and delta-compresses those ~1.94 GB of raw blobs down to the **989 MiB packfile** (`pack-….pack`) you actually download during `git clone`. Compression ratio is mediocre because APKs / `.tar.gz` / `.zip` are *already* compressed — git's zlib pass cannot squeeze much more out of them, and binary deltas between versions of an APK are weak.

### Why so many blobs under `releases/`?

The repo policy is "ship prebuilt binaries directly in the repo for users who can't reach GitHub Releases" (see `releases/README.md`). Each release adds a new set of files like:

- `mhrv-rs-android-universal-vX.Y.Z.apk` (~40 MB each)
- `mhrv-rs-android-arm64-v8a-vX.Y.Z.apk` (~19 MB)
- `mhrv-rs-android-armeabi-v7a-vX.Y.Z.apk` (~16 MB)
- `mhrv-rs-android-x86_64-vX.Y.Z.apk` (~19 MB)
- `mhrv-rs-android-x86-vX.Y.Z.apk` (~19 MB)
- `mhrv-rs-linux-amd64.tar.gz` (~8 MB)
- `mhrv-rs-linux-arm64.tar.gz` (~2 MB)
- `mhrv-rs-linux-musl-amd64.tar.gz` (~2 MB)
- `mhrv-rs-linux-musl-arm64.tar.gz` (~2 MB)
- `mhrv-rs-raspbian-armhf.tar.gz` (~2 MB)
- `mhrv-rs-openwrt-mipsel-softfloat.tar.gz` (~2 MB)
- `mhrv-rs-macos-amd64.tar.gz`, `mhrv-rs-macos-arm64.tar.gz` (~6–7 MB each)
- `mhrv-rs-macos-amd64-app.zip`, `mhrv-rs-macos-arm64-app.zip` (~5 MB each)
- `mhrv-rs-windows-amd64.zip` (~7 MB)

That's roughly **150 MB of new binaries per release**, and the following versions have all been committed at some point in history:

```
v1.0.0  v1.0.1  v1.0.2  v1.1.0
v1.7.6  v1.7.7  v1.7.8  v1.7.9  v1.7.11
v1.8.0  v1.8.1  v1.8.2
```

12 versions × ~150 MB raw ≈ **~1.8 GB of binary blobs** added to history over time. Even though the working tree only contains the latest version (157 MB), the older APKs and tarballs are still inside `.git/objects/pack` because git keeps every reachable object.

> Note: the working tree shows only ~16 release artifacts because each new release *replaces* the previous version's files at the same paths. The replaced blobs aren't deleted from history — they're orphaned in the packfile, which is why the pack is ~6× larger than the current `releases/` folder.

## 3. Why the upstream Python project is ~1 MB

`masterking32/MasterHttpRelayVPN` is a pure-Python project. Its repo contains only:

- Python source files (`.py`)
- An `apps_script/Code.gs` (a few KB of JavaScript pasted into Google Apps Script)
- A README + a couple of small assets

Python is interpreted, so there is **nothing to prebuild**. Users install dependencies with `pip install cryptography h2` and run the script. The repo therefore never accumulates per-platform binaries, and a `git clone` is on the order of **~1 MB**.

This Rust port made a different deployment choice (called out in the top-level `README.md`):

> "The original Python project is excellent but requires Python + `pip install …` + system deps. For users in hostile networks that install process is often itself broken (blocked PyPI, missing wheels, Windows without Python). This port is a single ~2.5 MB executable that you download and run."

To make that promise work for users who *also* can't reach GitHub Releases (blocked CDNs, censored networks), the maintainers commit the binaries straight into the repo so a plain `git clone` from any unblocked mirror is enough to obtain a runnable build. That convenience for end-users is exactly what costs you a 1 GB clone as a developer.

## 4. Summary of the size difference

| | Original Python project | This Rust port |
|---|---|---|
| Language | Python (interpreted) | Rust (compiled) |
| Per-platform artifacts in repo? | No | **Yes — all of them** |
| Number of binary artifacts in current tree | ~0 | ~16 (per release) |
| Number of historical release versions in `.git` | ~0 | 12 |
| Approximate clone size | ~1 MB | **~1 GB** |

The 1000× ratio is almost entirely accounted for by binary artifacts: ~1.92 GB of release blobs ever committed → ~989 MB packed inside `.git`.

## 5. What could be done to shrink the clone (informational only)

These are **not changes I am applying** — they would rewrite history and break every existing fork, pinned commit, and tag. Listed only because the user asked *why* the size is what it is.

1. **Stop committing binaries.** Ship them only via GitHub Releases / a CDN, like the Python project does for its few assets. New clones would drop by ~150 MB per release going forward, but existing history would still be heavy.
2. **Use Git LFS** for `releases/*.apk`, `releases/*.tar.gz`, `releases/*.zip`. New blobs go to LFS storage instead of the packfile; `git clone` becomes small, and LFS objects are pulled lazily on demand.
3. **Rewrite history to purge `releases/` from the past** with `git filter-repo --path releases/ --invert-paths` (or BFG Repo-Cleaner). This is the only way to get the existing ~989 MB packfile down to ~25 MB. It rewrites every commit hash, so it is a coordinated operation — every contributor has to re-clone, every external link to a commit SHA breaks, and force-push is required.
4. **Shallow / partial clone as a workaround for users:** `git clone --depth=1` only fetches the tip, skipping most historical blobs (still pulls the ~157 MB current `releases/`). `git clone --filter=blob:none` defers blob downloads until checkout. Neither changes the server-side repo, only the client's bandwidth.

## 6. How to verify these numbers yourself

```bash
# total clone size and where it goes
du -sh . .git .git/objects/pack releases/

# git's own accounting
git count-objects -vH

# top 25 largest blobs ever committed (path included)
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob" {print $3, $4}' \
  | sort -n | tail -25

# total bytes of all blobs that have ever lived under releases/
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob" && $4 ~ /^releases\// {sum+=$3} END {printf "%.1f MB\n", sum/1024/1024}'
```

All numbers in this document came from running exactly those commands against the current `main` branch.
