# Why this cloned repository is ~1 GB

## Summary

The repository clone is large because Git history contains many committed binary release artifacts (APK/ZIP/TAR.GZ files), mostly under `releases/`.  
Even if the current source code is small, a normal `git clone` downloads the full reachable object history by default.

## Evidence collected in this repository

- Working tree total size: about `1.2G`
- `.git` directory size: about `990M`
- Git pack size (`git count-objects -vH`): about `989.06 MiB`
- Largest blobs in history are release binaries, for example:
  - `releases/mhrv-rs-android-universal-v1.8.2.apk` (~40.8 MB)
  - `releases/mhrv-rs-android-universal-v1.7.11.apk` (~40.7 MB)
  - many Linux/Windows/macOS release archives in the 6-9 MB range each
- Aggregated historical blob size for `releases/` across all commits:
  - `release_blob_count=317`
  - `release_blob_total_bytes=2012194471` (~2.01 GB uncompressed blob data before pack/dedup)

## Why this differs from a small Python project (~1 MB)

A typical small Python project mainly contains text source files. Git compresses text very efficiently, so clone size often stays small.

This repository includes many binary build outputs over time:

- binaries are much larger than source files
- binaries delta-compress poorly compared to text
- each additional release artifact adds more historical object weight

So the clone reflects **code + full artifact history**, not just current code size.

## Practical ways to keep clone size smaller going forward

1. Stop committing release binaries into the main Git history.
2. Publish release assets via GitHub Releases (or artifact storage), not regular commits.
3. Add ignore rules for local build artifacts so they are not accidentally committed.
4. Use Git LFS for large files that must be tracked in Git workflows.
5. If desired later, rewrite history to remove old large blobs (for example with `git filter-repo`), then coordinate a forced migration for all clones.

## Notes

This document explains the current state and root cause.  
No history-rewrite action was performed in this change.
