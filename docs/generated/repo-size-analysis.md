# Repository Size Analysis

This document explains why cloning this repository can take almost **1 GB**, even though the original Python project is much smaller.

## Short answer

The large clone size is **not** coming from the Rust or Android source code.

It is coming mostly from:

1. the repository's **Git history** (`.git`)
2. many committed **prebuilt release artifacts** in `releases/`
3. large binary files such as **Android APKs**, `.zip`, and `.tar.gz` archives being stored across multiple versions

Git clones the history, not just the latest source snapshot, so old binary releases still make the clone heavy.

## Measurements from this checkout

These numbers were collected from the repository state in this branch:

| Path / Metric | Size |
| --- | ---: |
| Entire checkout on disk | `1.2G` |
| `.git` directory | `990M` |
| `releases/` in the current working tree | `157M` |
| `src/` | `652K` |
| `android/` | `432K` |
| `docs/` | `296K` |
| `assets/` | `64K` |

The important detail is that **almost all of the size is inside `.git`**, not in the editable source tree.

## Strong evidence from Git object storage

Git reports:

- packed objects: `2273`
- pack size: `989.06 MiB`

When summing the historical blob payload across all revisions:

- total blob payload across history: `1941.61 MiB`
- blob payload under `releases/`: `1918.98 MiB`

That means about **98.8%** of the historical file payload comes from `releases/`.

## What is making `releases/` so heavy

The largest historical blobs are release binaries such as:

- `releases/mhrv-rs-android-universal-v1.8.2.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.8.0.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.8.1.apk` -> `38.87 MiB`
- `releases/mhrv-rs-android-universal-v1.7.11.apk` -> `38.80 MiB`
- many more Android APKs in the `18-39 MiB` range

There are also multiple commits that refresh these release files, for example:

- `chore(releases): refresh prebuilt binaries for v1.8.2`
- `chore(releases): refresh prebuilt binaries for v1.8.1`
- `chore(releases): refresh prebuilt binaries for v1.8.0`
- older release refresh commits as well

So even if the current source code is small, the repository history still contains many generations of large binary artifacts.

## Why the original Python project can be much smaller

A Python-first project is often small because it is mostly:

- `.py` source files
- small config files
- documentation
- lightweight assets

Text files compress very well and usually stay tiny in Git history.

If the original Python repository did **not** check in compiled release outputs, APKs, archives, or many large generated binaries, then it would remain close to the size of its source code.

In other words:

- **Python project size** ~= source code size
- **this repository clone size** ~= source code size + checked-in release binaries + full history of those binaries

## Why Git handles this badly

Git is excellent for source code, but large versioned binaries are expensive because:

1. every changed binary becomes another large blob in history
2. files like `.apk`, `.zip`, and `.tar.gz` are already compressed
3. already-compressed files do not delta-compress nearly as well as text files

So repeated release updates quickly bloat `.git`.

## Practical conclusion

The repository is close to **1 GB** mainly because:

- the current checkout includes `releases/`
- the Git history includes many previous versions of those release artifacts
- the release artifacts are large compressed binaries, especially Android APKs

The actual source code footprint is small by comparison.

## If you want to reduce repository size later

Possible cleanup options:

1. **Stop committing release binaries to the repository**
   - publish them through GitHub Releases or CI artifacts instead
2. **Keep only metadata in Git**
   - checksums, release notes, manifests
3. **Rewrite history**
   - remove `releases/` from Git history using `git filter-repo` or BFG
   - this is the only way to materially shrink future clone size
4. **Use shallow clones as a temporary mitigation**
   - `git clone --depth 1 ...`

## Bottom line

The clone is big because this repository stores **release binaries in Git history**, not because the actual program source is large.

The original Python project was much smaller because it was likely mostly source files, while this repository carries a long history of compiled release artifacts.
