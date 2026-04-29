# Why Is This Repository ~1 GB When the Original Python Project Is ~1 MB?

## Short Answer

The repository stores **compiled binary releases for every supported platform, updated on every version bump, directly inside Git history**. Because Git keeps every past version of every file forever, ~30 rounds of binary replacements have accumulated to fill almost 1 GB of pack data — even though the working tree itself is only ~157 MB of binaries right now.

---

## The Original Python Project

[@masterking32's MasterHttpRelayVPN](https://github.com/masterking32/MasterHttpRelayVPN) is a pure-Python script. When you clone it you get:

- `.py` source files (plain text, tiny)
- A `requirements.txt`
- A README

The entire repository is plain text. Git stores text very efficiently (delta compression between versions), so the full clone with all history is well under 2 MB.

---

## What This Rust Port Added

This repository is a **Rust rewrite** (`mhrv-rs`) that compiles to standalone native binaries — no Python, no pip, no system dependencies. That design decision is intentional and solves a real problem for users on restricted networks:

> "This port is a single ~2.5 MB executable that you download and run. Nothing else."

To make those binaries available to users who cannot reach the GitHub Releases page (e.g., from inside a country with restricted internet), the maintainers committed the binaries **directly into the Git repository** under the `releases/` folder.

---

## What Causes the 1 GB Clone Size

### 1. Binary files stored in Git (the root cause)

The `releases/` folder contains 16 compiled artifacts for the current version:

| File | Size |
|---|---|
| `mhrv-rs-android-universal-v1.8.2.apk` | ~39 MB |
| `mhrv-rs-linux-amd64.tar.gz` | ~8.2 MB |
| `mhrv-rs-windows-amd64.zip` | ~6.9 MB |
| `mhrv-rs-macos-amd64.tar.gz` | ~6.6 MB |
| `mhrv-rs-macos-arm64.tar.gz` | ~6.0 MB |
| `mhrv-rs-macos-amd64-app.zip` | ~4.7 MB |
| `mhrv-rs-macos-arm64-app.zip` | ~4.4 MB |
| *(+ 9 more platform archives)* | |

**Working tree total for `releases/`: ~157 MB**

### 2. Git stores every historical version of every binary

Every time the maintainers cut a new release they run a step that **replaces all the files in `releases/` with the new version and commits**. Looking at the commit log, this has happened ~30 times:

```
chore(releases): refresh prebuilt binaries for v1.8.2
chore(releases): refresh prebuilt binaries for v1.8.1
chore(releases): refresh prebuilt binaries for v1.8.0
chore(releases): refresh prebuilt binaries for v1.7.11
chore(releases): refresh prebuilt binaries for v1.7.9
...  (≈ 30 total "refresh" commits)
```

Git is designed never to lose data. It keeps **every version** of every file as a separate "blob" object in its pack file. Even though you only *see* the latest binaries when you check out the repository, the `.git/` folder contains all ~30 generations of every binary.

### 3. Binaries do not compress or delta-compress well

Git uses zlib compression and delta encoding (storing only the *difference* between file versions) to keep history small for text files. Compiled binaries and APKs are already compressed internally (ZIP/zlib/LZ4/etc.), so:

- zlib re-compression gives **almost no benefit** — a 40 MB APK stays ~40 MB in the pack.
- Delta encoding between two different builds of the same binary also produces very little savings because compiled code changes are distributed unpredictably throughout the binary.

### 4. Measured evidence from this repository

Running `git verify-pack` on the pack file confirms this:

```
d3f922aa  blob  40,762,879 bytes  ← mhrv-rs-android-universal-v1.8.2.apk (current)
def028ec  blob  40,680,963 bytes  ← mhrv-rs-android-universal-v1.7.11.apk (old)
c48a3ce7  blob  17,347,903 bytes  ← android APK from an even older build
...
```

The `.git/objects/pack/` directory is **990 MB** total. The rest of the source code (Rust files, docs, configs) is a tiny fraction of that.

---

## Why Was This Choice Made?

The README explains the reasoning:

> "This folder contains the prebuilt binaries from the latest release, committed directly to the repository for users who cannot reach the GitHub Releases page."

The target audience is people behind restrictive firewalls (e.g., in Iran) who:
- Cannot reach `github.com/releases` (GitHub Releases is often blocked separately from the main site)
- **Can** reach the raw repository via `git clone` or a ZIP download

So the `releases/` folder is a deliberate distribution mechanism — a workaround for network-level censorship — not an oversight.

---

## Size Summary

| Component | Size |
|---|---|
| `.git/` pack (all history) | **990 MB** |
| `releases/` working tree | ~157 MB |
| `src/` (Rust source) | ~652 KB |
| `docs/`, `android/`, configs | < 1 MB |
| **Total clone size** | **~1.2 GB** |

The math is straightforward:

```
~30 release refreshes × ~16 binary files × average ~6 MB each
≈ ~2,880 MB of raw binary data stored across history
Compressed/delta-encoded in pack file → ~990 MB
```

---

## How to Get a Smaller Clone

If you only need the current code or binaries and not the full history, you have several options:

### Shallow clone (no history)
```bash
git clone --depth=1 https://github.com/therealaleph/MasterHttpRelayVPN-RUST.git
```
This downloads only the latest snapshot — reducing the clone to roughly **160–200 MB** instead of 1.2 GB.

### Sparse checkout (source only, skip `releases/`)
```bash
git clone --depth=1 --filter=blob:none --sparse \
  https://github.com/therealaleph/MasterHttpRelayVPN-RUST.git
cd MasterHttpRelayVPN-RUST
git sparse-checkout set src docs android tunnel-node assets
```
This gives you the source code and documentation without downloading any binary blobs — roughly **a few MB** total.

### Download only the binaries you need
Go directly to the [GitHub Releases page](https://github.com/therealaleph/MasterHttpRelayVPN-RUST/releases/latest) and download only the archive for your platform.

---

## How This Could Be Fixed (for the maintainers)

Storing large binary artifacts in Git history is considered an anti-pattern for exactly this reason. Alternative approaches:

1. **Use Git LFS (Large File Storage):** Git LFS stores binary blobs on a separate server and only keeps pointers in the Git history. Clone size stays small; the actual files are fetched on demand.

2. **Use GitHub Releases only:** Remove the `releases/` folder from the repository and always point users to the GitHub Releases page. For users who cannot reach it, provide a mirror link instead.

3. **BFG Repo Cleaner / `git filter-repo`:** The existing history can be rewritten to remove all old binary blobs, reducing the repository from ~1 GB back to a few MB. This is a destructive operation that rewrites commit SHAs and requires all collaborators to re-clone.

4. **GitHub Releases + a CDN mirror:** Keep the GitHub Releases workflow as-is but publish a separate mirror URL (e.g., a Cloudflare Pages or Bunny CDN endpoint) for censored users. No binaries ever enter the Git history.

---

## Conclusion

The ~1 GB clone size is **entirely explained by the `releases/` folder** — specifically by ~30 rounds of full binary replacement committed into Git history over the lifetime of the project. The original Python project has no compiled artifacts and is pure source text, so its history compresses to well under 2 MB. This Rust port made the deliberate trade-off of storing binaries in-repo to serve users on restricted networks, at the cost of a very large clone size.
