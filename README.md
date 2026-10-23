# onn. 4K Pro (2026, jarvis2) — root research

Status: **no root achieved.** This is a running log of what's been tried, what
worked, and what didn't, on the new 2026 onn. 4K Pro streaming box — codename
**jarvis2**, distinct from the earlier 2024 "jarvis/SNA" hardware that
[openensemble/onn-4k-pro-unlock-root](https://github.com/openensemble/onn-4k-pro-unlock-root)
already roots. That project's tooling explicitly excludes this device and
nothing in it transfers directly — different firmware, almost certainly a
different SoC generation, and (as far as we can tell) no public firmware dump
exists for jarvis2 anywhere yet.

If you're reading this because you have the same box: open an issue, or see
if [the upstream issue we filed](https://github.com/openensemble/onn-4k-pro-unlock-root/issues/1)
has moved. We'd rather consolidate than fork effort.

## Device identification

```
Retail name:   onn. 4K Pro Streaming Device (2026)
Codename:      jarvis2 (ro.product.device / .name / .board all agree)
Fingerprint:   onn/jarvis2/jarvis2:14/URO3.260203.035.A1/15640252:user/release-keys
Board platform: s7d  (not confirmed against a known public Amlogic part number)
Security patch: 2026-06-01
Active slot:   _a
ABI:           armeabi-v7a
ro.debuggable: 0  (production build; `adb root` refused)
ro.boot.flash.locked:        1
ro.boot.vbmeta.device_state: locked
ro.boot.verifiedbootstate:   green
sys.oem_unlock_allowed:      1  — Settings > Developer options > OEM unlocking
                                   shows ENABLED, not greyed out, on this unit
```

Also seen in the wild for this product line: model numbers DS9620 / JS620K4.

## Partition layout (A/B, dynamic partitions)

Read via `adb shell ls -la /dev/block/by-name/` — no root needed to list, but
every block device is `brw-------` (root-only to read):

```
boot_a/b, init_boot_a/b, dtbo_a/b, vendor_boot_a/b,
vbmeta_a/b, vbmeta_system_a/b, bootloader_a/b, tee, super, userdata, ...
```

Notably this device has real, non-empty-looking `init_boot_a`/`init_boot_b`
partitions separate from `boot_a`/`boot_b` — standard post-Android-13 GKI
ramdisk split. The jarvis/SNA hardware openensemble documented found its
`init_boot` partitions blank and roots `boot_b` instead, which is one
concrete sign jarvis2 is a newer/different kernel packaging than jarvis/SNA,
not just a renamed respin.

## Vendor interface

`com.droidlogic` and `com.android.networkstack.tethering.droidlogic` are
both present — the same Amlogic vendor package family openensemble's
HOW_IT_WORKS.md describes talking to for the AVB unlock on jarvis/SNA. This
is circumstantial, not proof: it tells us the same *class* of mechanism
likely exists here, not that any of their exact addresses, TEE trusted-app
UUID, or payload bytes apply. Those were hand-calibrated against a jarvis/SNA
firmware dump they had and we don't.

## Attempts so far

### 2026-10-23 — OTA capture attempt: dead end

Goal: trigger a system update check, catch the request to see what URL/host
the device talks to for its OTA payload, and (if the URL is a public,
unauthenticated blob like Google's standard OTA distribution pattern) fetch
it directly with curl from a computer — no device modification, no risk.

Opened `android.settings.SYSTEM_UPDATE_SETTINGS` via adb, confirmed via
`adb shell screencap`:

> **"Your system is up to date." Last successful check for update at 2:43 AM.**

No pending update means nothing to capture a URL for right now. This also
means the currently-*installed* firmware (what we actually want) was never
going to surface this way regardless — a "check for update" only ever reveals
the *next* release, not the one already on the box. The box being this
current is also a plausible reason nobody has a jarvis2 dump yet: there may
not have been a second OTA push since launch for anyone to have caught.

**Conclusion:** this path is blocked until Walmart/ONN ships a jarvis2 OTA
update. When one lands, re-attempt this — ideally with a MITM setup (user CA
cert installed via Settings, no root needed, plus a proxy on a LAN computer)
so the actual payload.bin can be captured even over HTTPS, not just the
destination host.

## What hasn't been tried yet

- Physical USB/fastboot test. Toggle shows OEM-unlock enabled, but whether
  `fastboot flashing unlock` is even implemented on this bootloader (vs.
  Amlogic's non-fastboot "burn mode"/storeboot USB-drive approach
  openensemble actually used) is unverified. Untested because an unlock
  attempt is potentially data-destructive and we wanted the firmware
  situation understood first.
- Reading `env`/`misc` partitions for U-Boot environment clues — blocked,
  same root-only permission issue as every other raw partition.
- Monitoring for the next OTA release to this device.

## Credit

[openensemble/onn-4k-pro-unlock-root](https://github.com/openensemble/onn-4k-pro-unlock-root)
for the only real prior art in this product line, and for documenting their
method well enough that someone else could tell where it does and doesn't
apply.
