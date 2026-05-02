# gborges manifest

A standalone repo manifest for AOSP 16 on Raspberry Pi 4 / Pi 5. Replaces the
combination of:

- `repo init -u https://android.googlesource.com/platform/manifest -b android-16.0.0_r4`
- the Raspberry Vanilla local manifests in `.repo/local_manifests/`

with a single git repo that vendors both upstreams as folder snapshots, plus
gborges-owned projects on top.

## Layout

```
manifest/
├── default.xml             top-level — <include>s the three sources below
├── google/                 verbatim snapshot of platform/manifest
│   ├── default.xml             (1045 lines, AOSP base)
│   └── SOURCE.md               provenance + bump procedure
├── raspberry-vanilla/      verbatim snapshot of raspberry-vanilla/android_local_manifest
│   ├── manifest_brcm_rpi.xml   wired in
│   ├── manifest_utilities.xml  wired in
│   ├── remove_projects.xml     available, not wired (opt-in for shallow sync)
│   ├── README.md               upstream's README, kept for reference
│   └── SOURCE.md               provenance + bump procedure
└── gborges.xml             gborges-owned projects (GB-AAOS/*)
```

`google/` and `raspberry-vanilla/` are byte-identical copies of their
upstreams as of the snapshot commit recorded in each folder's `SOURCE.md`.
`gborges.xml` is first-party and lives at the root.

## Include order

`default.xml` includes:

1. `google/default.xml` — AOSP base, must come first (provides the projects
   the next layer overrides).
2. `raspberry-vanilla/manifest_brcm_rpi.xml` — `<remove-project>` + project
   overrides for build/, camera, ffmpeg, graphics, etc.
3. `raspberry-vanilla/manifest_utilities.xml` — Pi-utils + V4L-utils.
4. `gborges.xml` — last, so it can override anything above.

`remove_projects.xml` is commented out in `default.xml`. Uncomment it to
strip unused `device/amlogic`, `device/linaro`, and similar projects from
the sync — matches the upstream raspberry-vanilla shallow-clone recipe.

## Reproducibility model

Each snapshot folder is a verbatim copy of upstream at a specific commit.
Inside the snapshot XMLs, project revisions are floating branches
(`revision="android-16.0"` etc.). **Reproducibility lives at the
manifest-repo git layer**: a given commit of this manifest repo always
points at the same snapshot files, but `repo sync` against those XMLs
will pull the current HEAD of the branches they name.

If you need stricter pinning (every project locked to a specific commit
SHA so re-syncs never drift), replace branch names with SHAs inside the
snapshot XMLs after copying them in. Trade-off: you lose the
"verbatim upstream" property and have to do the SHA-rewrite step on every
bump.

## Using it

Publish this directory as a git repo (any host), then on a fresh tree:

```sh
repo init -u <manifest-repo-url> -b <branch>
repo sync -j$(nproc)
```

Local testing without publishing:

```sh
repo init -u file:///mnt/build/aosp/manifest -b main
```

(requires the manifest dir to be a git repo with at least one commit on the
named branch — already initialised here on `main`).

## Bumping

- **AOSP base version** — see `google/SOURCE.md`. Re-clone
  `platform/manifest`, check out the new tag, copy `default.xml` over,
  update the SHA/date/tag rows in `SOURCE.md`.
- **Raspberry Vanilla** — see `raspberry-vanilla/SOURCE.md`. Re-clone the
  upstream repo, check out `android-16.0`, copy the four files over,
  update the SHA/date/subject rows in `SOURCE.md`.
- **gborges** — edit `revision="..."` in `gborges.xml` directly. Tracks
  `main` of `GB-AAOS/device_gborges` (and any future GB-AAOS projects
  added to that file).

After any bump, commit the manifest repo and re-`repo sync` consumers.
