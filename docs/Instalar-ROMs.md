# Instalar ROMs desde OrangeFox

Las ROMs **no necesitan nada especial** para instalarse desde OrangeFox. Si
una ROM instala en TWRP, instala aqui.

## Desde linea de comandos, sin tocar la pantalla

```bash
adb push ROM.zip /data/media/0/rom.zip
adb shell twrp install /data/media/0/rom.zip
adb shell twrp format data          # instalacion limpia
```

`twrp` es la herramienta de openrecoveryscript incluida. `twrp --help` lista
todo lo que acepta: `install`, `wipe`, `format data`, `sideload`, `decrypt`,
`mount`, `backup`, `restore`, `set_active`...

## Vendor R contra vendor S

**El error mas caro de diagnosticar.** Una ROM de vendor R instalada sobre un
telefono con firmware S se instala sin un solo error y luego da bootloop.

Si pasa eso, antes de culpar al recovery, verifica la instalacion. Ejemplo
real de una EvolutionX A16 que daba bootloop:

| Comprobacion | Resultado |
|---|---|
| system/vendor/product/system_ext en super | contenido correcto |
| `boot.img` en la particion | md5 **identico** al del zip |
| `dtbo.img` | md5 identico |
| `vbmeta` / `_system` / `_vendor` | identicas, con verificacion y hashtree desactivadas |
| `/metadata` (md_udc) | ext4 valido |

Todo perfecto. El problema era simplemente que la ROM era **R vendor**.

Comprobar la version del vendor instalado:

```bash
adb shell "mkdir -p /v; mount -o ro /dev/block/mapper/vendor /v; \
           grep ro.vendor.build.version.release /v/build.prop; umount /v"
```

## Comprobar que un flasheo fue bien

```bash
# la particion boot contra el zip
adb shell "dd if=/dev/block/by-name/boot bs=1M count=64 2>/dev/null | md5sum"

# flags AVB de vbmeta (0x3 = verificacion y hashtree desactivadas)
adb shell "dd if=/dev/block/by-name/vbmeta bs=4096 count=1 2>/dev/null" > vbmeta.bin
python3 -c "
import struct
b=open('vbmeta.bin','rb').read()
print('magic', b[:4], 'flags', hex(struct.unpack('>I', b[120:124])[0]))"
```

## El firmware

Algunos zips de ROM reflashean tambien el firmware (lk, tee, scp, spmfw,
sspm, md1img y el preloader). Eso es util: repara restos de una ROM de vendor
distinto. Se ve en la salida de la instalacion:

```
Patching lk image unconditionally...
Patching tee1 image unconditionally...
Patching md1img image unconditionally...
```

## La ROM te sobrescribe el recovery

Si la ROM lleva `install-recovery` activo, **cada arranque** restaura su propio
recovery:

```bash
adb shell getprop persist.vendor.recovery_update
```

Opciones:

1. **Flash Current OrangeFox** desde el propio recovery
2. Flashear el zip de OrangeFox otra vez
3. Desactivarlo: `setprop persist.vendor.recovery_update false` (necesita root;
   se pierde al formatear `/data`, porque vive en
   `/data/property/persistent_properties`)
