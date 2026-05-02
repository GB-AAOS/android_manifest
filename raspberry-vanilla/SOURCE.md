# raspberry-vanilla/ snapshot

Verbatim copy of the Raspberry Vanilla local manifest, used to layer
Pi-specific projects on top of the AOSP base in `../google/`.

| field | value |
|---|---|
| upstream repo | https://github.com/raspberry-vanilla/android_local_manifest |
| branch        | `android-16.0` |
| snapshot commit | `fdb000589e7655748be4baf3787c7bb66f8f81f9` |
| snapshot date | 2025-12-03 |
| snapshot subject | "update to android-16.0.0_r4" |

## Files

| file | wired into `default.xml`? | purpose |
|---|---|---|
| `manifest_brcm_rpi.xml`   | yes | core Pi projects (device/brcm/*, vendor/brcm, replacements for build/, camera, ffmpeg, graphics, …) |
| `manifest_utilities.xml`  | yes | Raspberry Pi utils + V4L utils replacement |
| `remove_projects.xml`     | yes | strips unused device/* and prebuilt projects to shrink the sync; opt in by uncommenting the include in `../default.xml` |
| `README.md`               | n/a | upstream's project README, kept for reference (not parsed by repo) |

## Updating

```sh
git clone https://github.com/raspberry-vanilla/android_local_manifest /tmp/rv
git -C /tmp/rv checkout android-16.0
cp /tmp/rv/manifest_brcm_rpi.xml /tmp/rv/manifest_utilities.xml \
   /tmp/rv/remove_projects.xml /tmp/rv/README.md \
   manifest/raspberry-vanilla/
git -C /tmp/rv rev-parse HEAD                  # → update snapshot commit above
git -C /tmp/rv log -1 --format=%s              # → update snapshot subject
```

Then update the `snapshot commit`, `snapshot date`, and `snapshot subject`
rows in the table and commit. The branch row only changes when the
underlying Android version bumps (e.g. `android-17.0`).

The `revision="android-16.0"` attributes inside the XMLs are floating
branch references — `repo sync` will pull HEAD of `android-16.0` for each
project. Reproducibility lives at the manifest-repo git layer (the SHA of
the manifest commit), not at the per-project layer.
