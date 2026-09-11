# USB, MTP y adb

El device tree original solo definia una regla de USB:

```
on property:sys.usb.config=adb && property:sys.usb.configfs=1
```

Pero el recovery pide varios modos. En `bootable/recovery`:

| Modo | Donde |
|---|---|
| `adb` | tenia regla |
| `mtp,adb` | **faltaba** |
| `mass_storage,adb` | **faltaba** |
| `sideload` | **faltaba** |
| `none` | existia, pero incompleta |

Sin regla, el gadget se queda **sin UDC**: adb muere y MTP no aparece.

## El cuelgue de `adb sideload`

```cpp
static bool SetUsbConfig(const std::string& state) {
    android::base::SetProperty("sys.usb.config", state);
    return android::base::WaitForProperty("sys.usb.state", state);
}
```

`WaitForProperty` **bloquea**. La regla `none` no publicaba `sys.usb.state`,
asi que al terminar el sideload el recovery se quedaba colgado para siempre en
"closing adb sideload". Ahora todas las reglas publican `sys.usb.state`.

La regla `none` ademas quitaba los enlaces de funciones **sin desatar antes el
UDC**, lo que tumba el gadget. Ahora escribe `UDC = "none"` primero.

## MTP: TWRP usa functionfs, no `mtp.gs0`

El MTP de TWRP es el de **functionfs** (`libtwrpmtp-ffs.so`), que necesita
`/dev/usb-ffs/mtp/ep0`. El tree solo montaba el de adb, asi que:

```
E [MTP] Failed to start usb driver!
```

Hace falta crear la instancia **antes** de montar el functionfs; al reves el
mount falla en silencio:

```
mkdir /config/usb_gadget/g1/functions/ffs.mtp     # primero la instancia
...
mkdir /dev/usb-ffs/mtp 0770 shell shell
mount functionfs mtp /dev/usb-ffs/mtp rmode=0770,fmode=0660,uid=2000,gid=2000,no_disconnect=1
```

## Por que adb moria justo al activar MTP

Este fue el mas sutil. En `partitionmanager.cpp`:

```cpp
property_set("sys.usb.config", "none");
TWFunc::write_to_file("/config/usb_gadget/g1/idVendor", "18D1");
TWFunc::write_to_file("/config/usb_gadget/g1/idProduct", "4EE2");
property_set("sys.usb.config", "mtp,adb");
```

TWRP escribe `18D1:4EE2`, pero el manejador `none` de init corre **de forma
asincrona** y hace:

```
write /config/usb_gadget/g1/idVendor ${vendor.usb.vid}    # 0x0E8D (MediaTek)
```

Es una carrera: init devuelve el VID a `0x0E8D` **despues** de que TWRP haya
puesto el de Google. El telefono enumera como `0E8D:4EE2`, un VID/PID que el
driver de adb del PC no reconoce. Encajaba con los sintomas: **MTP si**
funcionaba (usa driver de clase generico) y **sideload tambien** (ese modo usa
`4EE7`), pero adb normal no.

La regla `mtp,adb` ahora fija `18D1:4EE2` ella misma.

## Comprobar

```bash
adb shell "getprop sys.usb.config; getprop sys.usb.state"
adb shell "cat /config/usb_gadget/g1/UDC"
adb shell "mount | grep functionfs"      # deben salir adb, fastboot y mtp
```
