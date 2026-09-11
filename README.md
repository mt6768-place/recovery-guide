# recovery-guide

Todo lo necesario para construir y entender **OrangeFox R12.0** en Xiaomi
**merlinx** (Redmi Note 9 / Redmi 10X 4G, MT6768) sobre una ROM de
**vendor S** / Android 17.

## La documentacion esta en la wiki

👉 **[Ir a la wiki](../../wiki)**

| Pagina | Contenido |
|---|---|
| [Compilar OrangeFox](../../wiki/Compilar-OrangeFox) | sync, parches, build, trampas |
| [Descifrado FBE](../../wiki/Descifrado-FBE) | keymaster S, PIN/patron y sin credencial |
| [USB, MTP y adb](../../wiki/USB-MTP-y-adb) | reglas del gadget, sideload, VID/PID |
| [Instalar ROMs](../../wiki/Instalar-ROMs) | particiones dinamicas, vbmeta, firmware |
| [Problemas conocidos](../../wiki/Problemas-conocidos) | lo que no tiene arreglo y por que |

## Repos que hacen falta

| Que | Donde | Rama |
|---|---|---|
| Device tree del recovery | [`recovery_device_xiaomi_merlinx`](https://github.com/mt6768-place/recovery_device_xiaomi_merlinx) | `recovery-12.1` |
| Scripts, manifests y parches | [`mt6768-place-guide`](https://github.com/mt6768-place/mt6768-place-guide/tree/recovery) | `recovery` |
| Resto del arbol | `gitlab.com/OrangeFox/sync` | `fox_12.1` |

El device tree del recovery es **un repo aparte** del de la ROM. Comparten la
ruta `device/xiaomi/merlinx` dentro de sus respectivos arboles, pero no tienen
nada que ver entre si.

## Compilar en tres ordenes

```bash
git clone https://github.com/mt6768-place/mt6768-place-guide -b recovery guia
bash guia/scripts/sync.sh ~/fox_12.1
bash guia/scripts/apply-patches.sh ~/fox_12.1
bash guia/scripts/build.sh ~/fox_12.1
```

## Que incluye el recovery

- Descifrado FBE con PIN, patron, contrasena **y sin credencial**
- MTP y adb **a la vez**; `adb sideload` que cierra correctamente
- Instalacion de ROMs completas con particiones dinamicas
- Magisk, AromaFM, addon init.d, borrado de **FRP**, lptools, nano, bash
- **Flash Current OrangeFox**
