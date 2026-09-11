# recovery-guide

Everything needed to build and understand **OrangeFox R12.0** on the Xiaomi
**merlinx** (Redmi Note 9 / Redmi 10X 4G, MT6768) against an **S vendor** /
Android 17 ROM.

## Documentation

Full write-up in the [wiki](../../wiki), mirrored in [`docs/`](docs/):

| Page | Contents |
|---|---|
| [Index](docs/Index.md) | overview |
| [Building OrangeFox](docs/Building-OrangeFox.md) | sync, patches, build, traps |
| [FBE decryption](docs/FBE-decryption.md) | S-vendor keymaster, PIN/pattern, no credential |
| [USB, MTP and adb](docs/USB-MTP-and-adb.md) | gadget rules, sideload, VID/PID |
| [Installing ROMs](docs/Installing-ROMs.md) | dynamic partitions, vbmeta, firmware |
| [Known issues](docs/Known-issues.md) | what has no clean fix, and why |

## Repositories involved

| What | Where | Branch |
|---|---|---|
| Recovery device tree | [`recovery_device_xiaomi_merlinx`](https://github.com/mt6768-place/recovery_device_xiaomi_merlinx) | `recovery-12.1` |
| Scripts, manifests, patches | [`mt6768-place-guide`](https://github.com/mt6768-place/mt6768-place-guide/tree/recovery) | `recovery` |
| Everything else | `gitlab.com/OrangeFox/sync` | `fox_12.1` |

The recovery device tree is **a separate repository** from the ROM one. They
share the path `device/xiaomi/merlinx` inside their respective trees but are
unrelated.

## Build in three commands

```bash
git clone https://github.com/mt6768-place/mt6768-place-guide -b recovery guide
bash guide/scripts/sync.sh ~/fox_12.1
bash guide/scripts/apply-patches.sh ~/fox_12.1
bash guide/scripts/build.sh ~/fox_12.1
```

## What it ships

- FBE decryption with PIN, pattern, password **and no credential at all**
- MTP and adb **at the same time**; `adb sideload` that exits cleanly
- Installing full ROMs with dynamic partitions
- Magisk, AromaFM, init.d addon, **FRP** erase addon, lptools, nano, bash
- **Flash Current OrangeFox**

## Upstreamable fixes

Two of the patches are genuine bugs in upstream code, worth sending on their
own merit rather than carrying here forever:

- `system/vold`: an uninitialised token buffer and an unchecked AES-GCM tag in
  the FBE path
- `system/tools/aidl`: an uninitialised pointer that crashes on every
  parameterised annotation
