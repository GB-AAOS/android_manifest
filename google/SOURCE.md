# google/ snapshot

Verbatim copy of the AOSP platform manifest, used as the AOSP base for
this manifest.

| field | value |
|---|---|
| upstream repo | https://android.googlesource.com/platform/manifest |
| tag           | `refs/tags/android-16.0.0_r4` |
| snapshot commit | `15128c9e27cfa599c48d294babd39286ee8f1426` |
| snapshot date | 2025-12-02 |
| files in this dir | `default.xml` (1045 lines) |

## Updating

To bump to a newer AOSP release tag (e.g. `android-16.0.0_r5`):

```sh
git clone https://android.googlesource.com/platform/manifest /tmp/aosp-manifest
git -C /tmp/aosp-manifest checkout refs/tags/<new-tag>
cp /tmp/aosp-manifest/default.xml manifest/google/default.xml
git -C /tmp/aosp-manifest rev-parse HEAD       # → update snapshot commit above
```

Then update the `tag`, `snapshot commit`, and `snapshot date` rows in the
table, commit, and bump the matching reference in
`manifest/raspberry-vanilla/SOURCE.md` if the brcm branch needs to track
the new release.
