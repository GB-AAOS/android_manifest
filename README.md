# gborges manifest

A standalone repo manifest for AOSP 16 on Raspberry Pi 4 / Pi 5. Replaces the
combination of:

- `repo init -u https://android.googlesource.com/platform/manifest -b android-16.0.0_r4`
- the Raspberry Vanilla local manifests in `.repo/local_manifests/`

with a single git repo that vendors both upstreams as folder snapshots, plus
gborges-owned projects on top.

**Repo URL:** <https://github.com/GB-AAOS/android_manifest>
**Branch:** `main`

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
│   ├── remove_projects.xml     wired in (strips unused device/* + prebuilts)
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
4. `raspberry-vanilla/remove_projects.xml` — strips unused `device/amlogic`,
   `device/linaro`, etc. to shrink `repo sync`.
5. `gborges.xml` — last, so it can override anything above. Currently
   declares `device/gborges` and the nested `device/gborges/tonal_emulator`.

## Reproducibility model

Each snapshot folder is a verbatim copy of upstream at a specific commit.
Inside the snapshot XMLs, project revisions are floating branches
(`revision="android-16.0"` etc.). **Reproducibility lives at the
manifest-repo git layer**: a given commit of this manifest repo always
points at the same snapshot files, but `repo sync` against those XMLs
will pull the current HEAD of the branches they name.

If you need stricter pinning (every project locked to a specific commit
SHA so re-syncs never drift), replace branch names with SHAs inside the
snapshot XMLs after copying them in.

## Build

### 1. Prerequisites (Ubuntu 22.04+)

Set up the [Android build environment](https://source.android.com/docs/setup/start/requirements),
then add:

```sh
sudo apt-get install dosfstools e2fsprogs fdisk kpartx mtools rsync
```

### 2. Initialise and sync

```sh
mkdir aosp && cd aosp
repo init -u https://github.com/GB-AAOS/android_manifest -b main
repo sync -j$(nproc)
```

### 3. Lunch a target

```sh
source build/envsetup.sh
lunch gbrpi4_car-bp4a-userdebug      # or any target from the table below
```

| target                          | board / device | variant | build ID |
|--------------------------------|----------------|---------|----------|
| `gbrpi4_car-bp4a-userdebug`     | Pi 4 (hardware) | Automotive | bp4a |
| `gbrpi5_car-bp4a-userdebug`     | Pi 5 (hardware) | Automotive | bp4a |
| `tonal_emulator-bp4a-userdebug` | x86_64 emulator | Automotive demo with fake proximity sensor + monitor app | bp4a |

Build ID `bp4a` matches AOSP's `android-16.0.0_r4` release. The variant
suffix can be `user`, `userdebug`, or `eng`.

### 4. Compile

The build & deploy flow diverges per target type.

#### Pi 4 / Pi 5 hardware (`gbrpi4_car`, `gbrpi5_car`)

```sh
make bootimage systemimage vendorimage -j$(nproc)
```

`bootimage`, `systemimage`, and `vendorimage` are the three partition
images the flashable `.img` builder needs. A full `make -j$(nproc)` also
works but takes longer and produces nothing the SD-card image can use that
the three partition targets don't already cover.

Then package and flash:

```sh
./rpi4-mkimg.sh             # writes a flashable .img into ${ANDROID_PRODUCT_OUT}
./rpi4-wrimg.sh             # boot + system + vendor (probes /dev/sd[a-f])
./rpi4-wrimg.sh boot        # single partition
./rpi4-wrimg.sh wipe        # reformat metadata + userdata
```

`rpi5-mkimg.sh` / `rpi5-wrimg.sh` are the Pi 5 equivalents. The mkimg
scripts are not interchangeable — pick the one that matches the lunched
board. The wrimg scripts probe the SD card layout (boot=128M,
system=3072M, vendor=384M, metadata=16M) and bail out if no matching
device is found, so they are safe to run interactively.

See `device/gborges/README.md` for board-specific config (CAN bus on
Waveshare RS485 CAN HAT, GPIO via libgpiod, board/variant split, etc.).

#### Emulator (`tonal_emulator`)

Emulator targets do not produce SD-card partition images — build the
whole tree, then launch under QEMU:

```sh
m -j$(nproc)
emulator
```

The `tonal_emulator` device adds a fake proximity sensor (Sensors
Multi-HAL 2.1, sinusoidal output) and a pre-installed monitoring app on
top of an Automotive base. Useful runtime hooks:

```sh
adb shell setprop vendor.proximity.override 0.7   # force sensor value
adb shell setprop vendor.proximity.override -1.0  # back to sinusoidal
adb shell dumpsys sensorservice | grep -A 10 "Proximity Fake Sensor"
adb logcat -s ProximityMonitorApp
```

See `device/gborges/tonal_emulator/README.md` for sensor wiring,
SEPolicy notes, and the design document.

## Bumping

- **AOSP base version** — see `google/SOURCE.md`. Re-clone
  `platform/manifest`, check out the new tag, copy `default.xml` over,
  update the SHA/date/tag rows in `SOURCE.md`.
- **Raspberry Vanilla** — see `raspberry-vanilla/SOURCE.md`. Re-clone the
  upstream repo, check out `android-16.0`, copy the four files over,
  update the SHA/date/subject rows in `SOURCE.md`.
- **gborges** — edit `revision="..."` in `gborges.xml` directly. Each
  GB-AAOS project tracks `main`.

After any bump, commit the manifest repo and re-`repo sync` consumers.
