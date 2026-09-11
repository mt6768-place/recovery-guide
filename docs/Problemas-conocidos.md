# Problemas conocidos

## La UI va a 10-15 fps

**Sin arreglo viable.** Es molesto pero no afecta a la funcionalidad.

El backend grafico es fbdev; no hay DRM en este kernel:

```
Skipping adf graphics -- not present in build tree
cannot find/open a drm device: No such file or directory
Using fbdev graphics.
```

En cada frame, `set_displayed_framebuffer()` llama a `FBIOPUT_VSCREENINFO`,
que en el driver de MediaTek **no es un pan sencillo sino un mode-set completo
del pipeline de display**. De ahi los 10-15 fps.

### Por que no vale el buffer simple

Lo logico seria `RECOVERY_GRAPHICS_FORCE_SINGLE_BUFFER := true`, que elimina
ese ioctl por frame. **Se probo y deja la pantalla en negro.**

La razon esta en el propio codigo:

```c
static void set_displayed_framebuffer(unsigned n) {
    if (n > 1 || !double_buffered) return;     // <-- sale sin hacer nada
```

Con buffer simple la funcion no hace nada, asi que `vi.yoffset` nunca se pone
a 0 y el panel se queda mostrando una zona del framebuffer donde no dibujamos.
Resultado: recovery funcionando y accesible por adb, pero pantalla negra.

Se podria forzar `yoffset` a mano, pero es tocar el backend grafico a ciegas
con riesgo alto de dejar el dispositivo sin pantalla.

## `Unable to unlock /dev/block/mmcblk0: Permission denied`

Cosmetico. El recovery recorre **todos** los dispositivos de bloque quitando
el flag de solo-lectura con `BLKROSET`, y el eMMC crudo lo rechaza. El bucle
hace `continue` y no pasa nada.

Se baja a informativo con `export OF_LOOP_DEVICE_ERRORS_TO_LOG=1` en
`vendorsetup.sh`.

## KernelSU

`FOX_ENABLE_KERNELSU_SUPPORT` **rompe la build** en este dispositivo:

```
FOX_ENABLE_KERNELSU_SUPPORT is only valid for Virtual_AB devices
with a GKI 5.x or 6.x kernel
```

merlinx es A-only con kernel 4.14/4.19. No aplica. Para root, usa el Magisk
que ya viene incluido.

## Reservado a vendor S

Este recovery lleva el keymaster de **vendor S**. Sobre una ROM de vendor R el
descifrado puede no funcionar, aunque se arrancan ambos servicios (4.0 y 4.1) y
el manifiesto declara las dos versiones para maximizar la compatibilidad.

## Hay que usar tmux en servidores compartidos

En algunos servidores las builds largas mueren con SIGKILL si se lanzan
directamente por SSH. Lanzalas dentro de `tmux` o `screen`.
