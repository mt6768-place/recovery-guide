# Installing ROMs from OrangeFox

ROMs need **nothing special** to install from OrangeFox. If a ROM installs in
TWRP, it installs here.

## From the command line, without touching the screen

```bash
adb push ROM.zip /data/media/0/rom.zip
adb shell twrp install /data/media/0/rom.zip
adb shell twrp format data          # clean install
```

`twrp` is the bundled openrecoveryscript tool. `twrp --help` lists everything
it accepts: `install`, `wipe`, `format data`, `sideload`, `decrypt`, `mount`,
`backup`, `restore`, `set_active`...

## R vendor versus S vendor

**The most expensive mistake to diagnose.** An R-vendor ROM installed on a
phone with S firmware installs without a single error and then bootloops.

If that happens, verify the installation before blaming the recovery. A real
example with an EvolutionX A16 build that bootlooped:

| Check | Result |
|---|---|
| system/vendor/product/system_ext in super | correct content |
| `boot.img` on the partition | md5 **identical** to the zip |
| `dtbo.img` | md5 identical |
| `vbmeta` / `_system` / `_vendor` | identical, verification and hashtree disabled |
| `/metadata` (md_udc) | valid ext4 |

Everything perfect. The problem was simply that the ROM was **R vendor**.

Checking the installed vendor version:

```bash
adb shell "mkdir -p /v; mount -o ro /dev/block/mapper/vendor /v; \
           grep ro.vendor.build.version.release /v/build.prop; umount /v"
```

## Verifying a flash

```bash
# boot partition against the zip
adb shell "dd if=/dev/block/by-name/boot bs=1M count=64 2>/dev/null | md5sum"

# vbmeta AVB flags (0x3 = verification and hashtree disabled)
adb shell "dd if=/dev/block/by-name/vbmeta bs=4096 count=1 2>/dev/null" > vbmeta.bin
python3 -c "
import struct
b=open('vbmeta.bin','rb').read()
print('magic', b[:4], 'flags', hex(struct.unpack('>I', b[120:124])[0]))"
```

## Firmware

Some ROM zips reflash the firmware as well (lk, tee, scp, spmfw, sspm, md1img
and the preloader). That is useful: it repairs leftovers from a ROM with a
different vendor. You can see it in the install output:

```
Patching lk image unconditionally...
Patching tee1 image unconditionally...
Patching md1img image unconditionally...
```

## The ROM overwrites your recovery

If the ROM has `install-recovery` active, **every boot** restores its own
recovery:

```bash
adb shell getprop persist.vendor.recovery_update
```

Options:

1. **Flash Current OrangeFox** from the recovery itself
2. Flash the OrangeFox zip again
3. Disable it: `setprop persist.vendor.recovery_update false` (needs root, and
   it is lost when `/data` is formatted because it lives in
   `/data/property/persistent_properties`)
