# OrangeFox for merlinx

Documentation for **OrangeFox R12.0** (TWRP 12.1 base) on the Xiaomi
**merlinx** (Redmi Note 9 / Redmi 10X 4G, MT6768 Helio G85), targeting a host
ROM with an **S vendor** and Android 17.

## Pages

- **[Building OrangeFox](Building-OrangeFox.md)** — sync, patches, build, and the
  two traps that cost the most time
- **[FBE decryption](FBE-decryption.md)** — why it failed and how it was fixed
- **[USB, MTP and adb](USB-MTP-and-adb.md)** — gadget rules and sideload
- **[Installing ROMs](Installing-ROMs.md)** — dynamic partitions, vbmeta, firmware
- **[Known issues](Known-issues.md)** — what has no clean fix, and why

## Why this exists

The public OrangeFox device tree for merlin targets an **R vendor** (MIUI 12).
Used against an **S vendor** ROM the recovery booted, but:

- it could not decrypt `/data` (the keymaster crashed with SIGSEGV)
- MTP did not work, and adb died the moment it was enabled
- `adb sideload` hung on exit
- the build did not even finish: it died at 99%

Each page covers one problem, how it was diagnosed and what the fix was. All
changes are published in this organisation.

## Worth knowing up front

If the ROM has `install-recovery` active it **overwrites the recovery on every
boot**:

```bash
adb shell getprop persist.vendor.recovery_update
```

To get it back: **Flash Current OrangeFox** from the recovery itself, or flash
the zip over fastboot.
