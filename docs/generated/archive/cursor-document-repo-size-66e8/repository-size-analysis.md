# Why this repository clone is almost 1 GB

This note explains why cloning this Rust port can use close to 1 GB on disk even though the original Python project is much smaller.

## Short answer

The size is not caused by Rust source code. It is caused mostly by release binaries that are committed into the repository and by Git keeping every historical version of those binaries.

In this checkout:

- Total repository directory size: about `1.2G`
- Git object database (`.git/objects`): about `990M`
- Current tracked files in `HEAD`: about `157.70 MiB`
- Current `releases/` directory: about `157M`

So most of the clone size comes from two places:

1. The current `releases/` folder, which contains prebuilt archives and APK files.
2. The `.git` history, which contains older versions of those same binary release files.

## What is large in the current checkout?

The source tree itself is small. The large files are prebuilt release artifacts under `releases/`.

Largest current tracked files include:

| File | Approx size |
|---|---:|
| `releases/mhrv-rs-android-universal-v1.8.2.apk` | 39 MiB |
| `releases/mhrv-rs-android-x86_64-v1.8.2.apk` | 19 MiB |
| `releases/mhrv-rs-android-x86-v1.8.2.apk` | 19 MiB |
| `releases/mhrv-rs-android-arm64-v8a-v1.8.2.apk` | 19 MiB |
| `releases/mhrv-rs-android-armeabi-v7a-v1.8.2.apk` | 16 MiB |
| `releases/mhrv-rs-linux-amd64.tar.gz` | 8.2 MiB |
| `releases/mhrv-rs-windows-amd64.zip` | 6.9 MiB |
| `releases/mhrv-rs-macos-amd64.tar.gz` | 6.6 MiB |

Together, the current release artifacts account for almost all of the `157.70 MiB` currently tracked by Git.

## Why Git history makes it much bigger

Git stores the current files plus the repository history. For normal text files, Git can compress changes very efficiently. For compiled binaries, APKs, `.zip`, and `.tar.gz` archives, Git cannot delta-compress them nearly as well.

This repository has many commits that refreshed the files in `releases/`, for example:

- `chore(releases): refresh prebuilt binaries for v1.8.2`
- `chore(releases): refresh prebuilt binaries for v1.8.1`
- `chore(releases): refresh prebuilt binaries for v1.8.0`
- `chore(releases): refresh prebuilt binaries for v1.7.11`
- `chore(releases): refresh prebuilt binaries for v1.7.9`
- `chore(releases): refresh prebuilt binaries for v1.7.8`
- `chore(releases): refresh prebuilt binaries for v1.7.7`
- `chore(releases): refresh prebuilt binaries for v1.7.6`

Each refresh adds or replaces large binary files. Even when a file name is reused, Git still keeps the old blob in history because older commits must remain reproducible.

Some of the largest blobs in history are Android universal APKs and release archives:

| Historical blob path | Approx uncompressed blob size |
|---|---:|
| `releases/mhrv-rs-android-universal-v1.8.2.apk` | 40.8 MiB |
| `releases/mhrv-rs-android-universal-v1.7.11.apk` | 40.7 MiB |
| `releases/mhrv-rs-android-universal-v1.1.0.apk` | 11.8 MiB |
| `releases/mhrv-rs-android-universal-v1.0.2.apk` | 17.3 MiB |
| `releases/mhrv-rs-linux-amd64.tar.gz` | 8.4-8.5 MiB per historical version |

Because there are many versions of these generated files, the packed Git database grows to about `989 MiB`.

## Why the original Python project can be around 1 MB

The original Python project is mostly source code and small support files. It does not need to commit compiled binaries for many platforms.

This Rust port, by contrast, includes ready-to-download release artifacts for:

- Android APK variants
- Linux archives
- macOS archives and app bundles
- Windows archives
- OpenWRT and Raspberry Pi archives

Those files are useful for users who want a direct download, but they are expensive inside Git history.

## Important distinction: source size vs clone size

The Rust source code is still relatively small. The big number appears when measuring a full Git clone because a full clone includes:

- The current working tree
- The `.git` directory
- Packed historical objects
- Every committed version of release binaries

That is why the project can have a small source code base while the clone takes nearly 1 GB.

## How to reduce clone size as a user

If you only need the latest source code and do not need full history, use a shallow clone:

```bash
git clone --depth 1 <repo-url>
```

This downloads only the latest commit. It avoids most historical binary blobs.

If you also do not need the checked-out release artifacts, a partial clone can help:

```bash
git clone --filter=blob:none <repo-url>
```

Git will download file contents lazily when needed. This is useful when browsing history or source code without immediately downloading every large blob.

## How the repository could reduce size long-term

For maintainers, the best long-term fix is to avoid committing generated release binaries to normal Git history.

Common options are:

1. Store release binaries only in GitHub Releases.
2. Use Git LFS for large binary artifacts.
3. Keep `releases/README.md`, but remove generated archives and APKs from tracked source.
4. If the old history must be reduced, rewrite repository history with a tool such as `git filter-repo` or BFG Repo-Cleaner, then force-push with careful coordination.

History rewriting is disruptive because every contributor must re-clone or repair local branches, so it should be planned carefully.

## Commands used for this analysis

```bash
du -h --max-depth=2 . | sort -h
git count-objects -vH
git ls-tree -r -l HEAD
git verify-pack -v .git/objects/pack/*.idx
git log --stat --oneline --all -- releases assets android
```
