# gborges manifest

A standalone repo manifest for AOSP 16 on Raspberry Pi 4 / Pi 5. Replaces the
combination of:

- `repo init -u https://android.googlesource.com/platform/manifest -b android-16.0.0_r4`
- the Raspberry Vanilla local manifests in `.repo/local_manifests/`

with a single, reproducible source.

## Files

- `default.xml` — top-level. `<include>`s the two below.
- `aosp.xml` — verbatim copy of `platform/manifest @ refs/tags/android-16.0.0_r4`
  (1045 lines). Re-copy from upstream when bumping the AOSP base version.
- `brcm.xml` — Raspberry Vanilla projects merged from upstream
  ([raspberry-vanilla/android_local_manifest](https://github.com/raspberry-vanilla/android_local_manifest))
  with every `revision="android-16.0"` replaced by a specific commit SHA.
  No floating branches — sync results are reproducible.

`aosp.xml` includes first; `brcm.xml`'s `<remove-project>` directives
depend on AOSP entries already being present.

## Using it

Publish this directory as a git repo (any host), then on a fresh tree:

```
repo init -u <manifest-repo-url> -b <branch>
repo sync -j$(nproc)
```

For local testing without publishing, point `repo init` at the directory:

```
repo init -u file:///mnt/build/aosp/manifest -b main
```

(requires the manifest dir to be a git repo with at least one commit on the
named branch).

## Bumping pinned SHAs

The SHAs in `brcm.xml` were captured from the working checkout on
2026-05-02. To advance to newer upstream:

```
# in each working copy under device/brcm/, external/, etc.
git fetch origin android-16.0
git rev-parse origin/android-16.0
```

Replace the matching `revision="..."` in `brcm.xml`, commit, push.
The `upstream="android-16.0"` attribute is informational — `repo sync`
honors `revision` exclusively.

## Bumping the AOSP base version

Re-copy from the upstream manifest repo:

```
curl -sLo aosp.xml https://android.googlesource.com/platform/manifest/+/refs/tags/<new-tag>/default.xml?format=TEXT | base64 -d
```

or `git clone https://android.googlesource.com/platform/manifest`,
checkout the desired tag, and copy `default.xml` into this dir.
