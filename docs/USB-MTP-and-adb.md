# USB, MTP and adb

The original device tree defined a single USB rule:

```
on property:sys.usb.config=adb && property:sys.usb.configfs=1
```

But the recovery asks for several modes. In `bootable/recovery`:

| Mode | Status |
|---|---|
| `adb` | had a rule |
| `mtp,adb` | **missing** |
| `mass_storage,adb` | **missing** |
| `sideload` | **missing** |
| `none` | present, but incomplete |

With no rule the gadget is left **with no UDC**: adb dies and MTP never
appears.

## The `adb sideload` hang

```cpp
static bool SetUsbConfig(const std::string& state) {
    android::base::SetProperty("sys.usb.config", state);
    return android::base::WaitForProperty("sys.usb.state", state);
}
```

`WaitForProperty` **blocks**. The `none` rule never published
`sys.usb.state`, so on finishing a sideload the recovery hung forever on
"closing adb sideload". Every rule now publishes `sys.usb.state`.

The `none` rule also removed the function symlinks **without detaching the UDC
first**, which tears the gadget down. It now writes `UDC = "none"` first.

## MTP: TWRP uses functionfs, not `mtp.gs0`

TWRP's MTP is the **functionfs** one (`libtwrpmtp-ffs.so`), which needs
`/dev/usb-ffs/mtp/ep0`. The tree only mounted the adb instance, so:

```
E [MTP] Failed to start usb driver!
```

The function instance must be created **before** mounting the functionfs; the
other way round the mount fails silently:

```
mkdir /config/usb_gadget/g1/functions/ffs.mtp     # instance first
...
mkdir /dev/usb-ffs/mtp 0770 shell shell
mount functionfs mtp /dev/usb-ffs/mtp rmode=0770,fmode=0660,uid=2000,gid=2000,no_disconnect=1
```

## Why adb died exactly when MTP was enabled

This was the subtlest one. In `partitionmanager.cpp`:

```cpp
property_set("sys.usb.config", "none");
TWFunc::write_to_file("/config/usb_gadget/g1/idVendor", "18D1");
TWFunc::write_to_file("/config/usb_gadget/g1/idProduct", "4EE2");
property_set("sys.usb.config", "mtp,adb");
```

TWRP writes `18D1:4EE2`, but init's `none` handler runs **asynchronously** and
does:

```
write /config/usb_gadget/g1/idVendor ${vendor.usb.vid}    # 0x0E8D (MediaTek)
```

It is a race: init restores the VID to `0x0E8D` **after** TWRP has set
Google's. The phone enumerates as `0E8D:4EE2`, a VID/PID the host's adb driver
does not recognise. That matched the symptoms exactly: **MTP worked** (generic
class driver) and **sideload worked** (that mode uses `4EE7`), but plain adb
did not.

The `mtp,adb` rule now pins `18D1:4EE2` itself.

## Verifying

```bash
adb shell "getprop sys.usb.config; getprop sys.usb.state"
adb shell "cat /config/usb_gadget/g1/UDC"
adb shell "mount | grep functionfs"      # adb, fastboot and mtp should appear
```
