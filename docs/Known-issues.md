# Known issues

## The UI runs at 10-15 fps

**No viable fix.** Annoying, but it does not affect functionality.

The graphics backend is fbdev; there is no DRM on this kernel:

```
Skipping adf graphics -- not present in build tree
cannot find/open a drm device: No such file or directory
Using fbdev graphics.
```

On every frame `set_displayed_framebuffer()` calls `FBIOPUT_VSCREENINFO`,
which on MediaTek's driver is **not a simple pan but a full mode-set of the
display pipeline**. Hence the 10-15 fps.

### Why single buffering does not work

The obvious move is `RECOVERY_GRAPHICS_FORCE_SINGLE_BUFFER := true`, which
removes that per-frame ioctl. **It was tried and it leaves the screen black.**

The reason is in the code itself:

```c
static void set_displayed_framebuffer(unsigned n) {
    if (n > 1 || !double_buffered) return;     // <-- returns doing nothing
```

With single buffering the function does nothing, so `vi.yoffset` is never set
to 0 and the panel keeps showing a region of the framebuffer we do not draw
into. Result: a recovery that runs and is reachable over adb, with a black
screen.

Forcing `yoffset` by hand is possible but means changing the graphics backend
blind, with a high risk of leaving the device with no display.

## `Unable to unlock /dev/block/mmcblk0: Permission denied`

Cosmetic. The recovery walks **every** block device clearing the read-only
flag with `BLKROSET`, and the raw eMMC refuses. The loop just continues.

Demote it to informational with `export OF_LOOP_DEVICE_ERRORS_TO_LOG=1` in
`vendorsetup.sh`.

## KernelSU

`FOX_ENABLE_KERNELSU_SUPPORT` **breaks the build** on this device:

```
FOX_ENABLE_KERNELSU_SUPPORT is only valid for Virtual_AB devices
with a GKI 5.x or 6.x kernel
```

merlinx is A-only with a 4.14/4.19 kernel, so it does not apply. For root, use
the bundled Magisk.

## Aimed at an S vendor

This recovery ships the **S-vendor** keymaster. On an R-vendor ROM decryption
may not work, although both services (4.0 and 4.1) are started and the manifest
declares both versions to maximise compatibility.

## Use tmux on shared servers

On some servers long builds are killed with SIGKILL when launched directly over
SSH. Run them inside `tmux` or `screen`.
