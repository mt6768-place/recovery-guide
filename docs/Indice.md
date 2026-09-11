# OrangeFox para merlinx

Documentacion de **OrangeFox R12.0** (base TWRP 12.1) en Xiaomi **merlinx**
(Redmi Note 9 / Redmi 10X 4G, MT6768 Helio G85), pensado para convivir con una
ROM de **vendor S** y Android 17.

## Paginas

- **[Compilar OrangeFox](Compilar-OrangeFox.md)** — sync, parches, build y las
  dos trampas que mas tiempo cuestan
- **[Descifrado FBE](Descifrado-FBE.md)** — por que fallaba y como se arreglo
- **[USB, MTP y adb](USB-MTP-y-adb.md)** — reglas del gadget y sideload
- **[Instalar ROMs](Instalar-ROMs.md)** — particiones dinamicas, vbmeta, firmware
- **[Problemas conocidos](Problemas-conocidos.md)** — lo que no tiene arreglo

## Por que existe esta documentacion

El device tree publico de OrangeFox para merlin estaba hecho para **vendor R**
(MIUI 12). Al usarlo sobre una ROM de **vendor S** el recovery arrancaba, pero:

- no descifraba `/data` (el keymaster crasheaba con SIGSEGV)
- MTP no funcionaba y adb moria al activarlo
- `adb sideload` se colgaba al cerrar
- la build ni siquiera terminaba: petaba al 99%

Cada pagina explica un problema, como se diagnostico y cual fue el arreglo.
Todos los cambios estan publicados en esta organizacion.

## Aviso importante

Si la ROM tiene `install-recovery` activo, **sobrescribe el recovery en cada
arranque**. Se comprueba asi:

```bash
adb shell getprop persist.vendor.recovery_update
```

Para recuperarlo: **Flash Current OrangeFox** desde el propio recovery, o
flashear el zip por fastboot.
