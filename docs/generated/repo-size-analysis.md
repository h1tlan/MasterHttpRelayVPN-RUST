# Why this repository clones at about 1 GB while the original Python project is much smaller

This document explains why a clone of this repository is about **1 GB** even though the upstream Python project is closer to **source-only size**.

## Short answer

The size is **not** coming from the Rust source code.

It comes mainly from:

1. the repository's **Git history** in `.git`
2. many committed **prebuilt release binaries** under `releases/`
3. repeated version updates of large compressed files such as `.apk`, `.zip`, and `.tar.gz`

Git clones history, not just the latest snapshot, so old release binaries still make the repository heavy.

## Measurements from this checkout

These numbers were measured from this repository:

| Path / Metric | Size |
| --- | ---: |
| Entire checkout on disk | `1.2G` |
| `.git` directory | `990M` |
| `releases/` in the current working tree | `157M` |
| `src/` | `652K` |
| `android/` | `432K` |
| `docs/` | `296K` |
| `assets/` | `64K` |

The key point is that **most of the size is in `.git`**, not in the editable source tree.

## Evidence from Git object storage

Git reports:

- packed objects: `2273`
- pack size: `989.06 MiB`

When historical blob payload is summed across all revisions:

- total blob payload across history: `1941.61 MiB`
- blob payload under `releases/`: `1918.98 MiB`

So about **98.8%** of the historical file payload comes from `releases/`.

## What is making the repository heavy

The current working tree already contains large prebuilt artifacts, including:

| File | Approx size |
| --- | ---: |
| `releases/mhrv-rs-android-universal-v1.8.2.apk` | `39 MiB` |
| `releases/mhrv-rs-android-x86_64-v1.8.2.apk` | `19 MiB` |
| `releases/mhrv-rs-android-x86-v1.8.2.apk` | `19 MiB` |
| `releases/mhrv-rs-android-arm64-v8a-v1.8.2.apk` | `19 MiB` |
| `releases/mhrv-rs-android-armeabi-v7a-v1.8.2.apk` | `16 MiB` |
| `releases/mhrv-rs-linux-amd64.tar.gz` | `8.2 MiB` |
| `releases/mhrv-rs-windows-amd64.zip` | `6.9 MiB` |
| `releases/mhrv-rs-macos-amd64.tar.gz` | `6.6 MiB` |

The largest historical blobs are also release binaries, for example:

- `releases/mhrv-rs-android-universal-v1.8.2.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.8.0.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.8.1.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.7.11.apk` -> `38.80 MiB`

There are many commits that refreshed release binaries, for example:

- `chore(releases): refresh prebuilt binaries for v1.8.2`
- `chore(releases): refresh prebuilt binaries for v1.8.1`
- `chore(releases): refresh prebuilt binaries for v1.8.0`
- `chore(releases): refresh prebuilt binaries for v1.7.11`
- older release refresh commits as well

Each refresh adds another generation of large binary blobs to history.

## Why the original Python project is much smaller

The upstream Python project is mostly:

- `.py` source files
- small support files
- documentation
- lightweight assets

Text source compresses very well in Git.

This Rust port made a different trade-off: it includes ready-to-download compiled artifacts for multiple platforms, including Android, Linux, macOS, Windows, OpenWRT, and Raspberry Pi builds. That is convenient for users, but expensive in Git history.

## Why Git size grows so fast with binaries

Git works very well for source code, but large versioned binaries are costly because:

1. every changed binary becomes another large blob in history
2. files like `.apk`, `.zip`, and `.tar.gz` are already compressed
3. already-compressed files do not delta-compress nearly as well as text files

That is why repeated release updates quickly bloat `.git`.

## Why this was likely done on purpose

The repository appears to keep prebuilt binaries in `releases/` so users can obtain runnable builds directly from the repository, even in environments where package installation or GitHub Releases downloads may be difficult.

That distribution choice helps end users, but it also makes developer clones much larger.

## Practical conclusion

The repository is close to **1 GB** mainly because:

- the current checkout includes `releases/`
- the Git history includes many previous versions of those release artifacts
- the release artifacts are large compressed binaries, especially Android APKs

The actual program source is small by comparison.

## Ways to reduce the size later

Possible cleanup options for maintainers:

1. **Stop committing release binaries to the repository**
   - publish them through GitHub Releases or another artifact channel instead
2. **Use Git LFS**
   - keep large binary artifacts outside the normal Git packfile flow
3. **Rewrite history**
   - remove `releases/` from Git history using `git filter-repo` or BFG
   - this is the only way to materially shrink future full clones
4. **Use shallow or filtered clones as a workaround**
   - `git clone --depth 1 ...`
   - `git clone --filter=blob:none ...`

## Bottom line

The clone is large because this repository stores **release binaries in Git history**, not because the source code itself is large.

The original Python project stays much smaller because it is mostly source files, while this repository carries a long history of compiled release artifacts.
