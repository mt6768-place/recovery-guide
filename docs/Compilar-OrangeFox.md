# Compilar OrangeFox

```bash
git clone https://github.com/mt6768-place/mt6768-place-guide -b recovery guia
bash guia/scripts/sync.sh ~/fox_12.1
bash guia/scripts/apply-patches.sh ~/fox_12.1
bash guia/scripts/build.sh ~/fox_12.1
```

La imagen y el zip instalable salen en `out/target/product/merlinx/`.

## Los cuatro parches

Viven en repos que no alojamos. Sin ellos, o la build falla, o el recovery
arranca pero no sirve. Los dos primeros son **bugs de memoria sin inicializar**
que solo se manifiestan porque Android compila con
`-ftrivial-auto-var-init=pattern`, que rellena las variables locales con
`0xAA...`.

### `system/tools/aidl` — la build petaba al 99%

```cpp
struct ConstReferenceFinder : AidlVisitor {
  const AidlConstantReference* found;   // sin inicializar
```

Vale `0xaaaaaaaaaaaaaaaa`, asi que `if (!found)` nunca se cumple, `Find()`
devuelve un puntero basura y `AIDL_ERROR()` lo desreferencia. SIGSEGV en
**toda anotacion AIDL con parametros**. Se arregla con `= nullptr`.

Se localizo ejecutando el `aidl` compilado bajo gdb: el fallo era reproducible
con un solo `.aidl` que tuviera una anotacion con parametros.

### `bootable/recovery` — `twres/` vacio

El tema se copia durante el **analisis de Soong**, no en una regla de ninja,
desde `gui/libguitwrp_defaults.go`:

```go
outDir := ctx.Config().Getenv("OUT")
twRes := outDir + "/recovery/root/twres/"
os.MkdirAll(twRes, os.ModePerm)      // error ignorado
copyDir(dirToCopy, destDir)          // error ignorado
```

Si `OUT` llega como ruta **relativa** y `soong_build` corre desde otro
directorio, el destino se resuelve mal, `MkdirAll` falla y **todos los errores
se descartan**. La build sigue tan campante y muere al 99%:

```
sed: .../recovery/root/twres/splash.xml: No such file or directory
```

El parche ancla `OUT` a ruta absoluta.

### `system/vold` — descifrado

Ver **[Descifrado FBE](Descifrado-FBE.md)**.

### `vendor/recovery` — builds que se pisan

`OrangeFox_A12.sh` usa rutas fijas en `/tmp` para su estado
(`/tmp/fox_build_000tmp.txt`, `/tmp/Fox_000_tmp`, `/tmp/oFox00.tmp`...). En una
maquina compartida, dos usuarios compilando OrangeFox a la vez **se leen las
variables el uno al otro**.

Sintoma real: la imagen salio a `/OrangeFox-...img`, con `$OUT` vacio, porque
el script cargo el fichero de estado de otro usuario que compilaba otro
dispositivo. El parche mete todo bajo `/tmp/ofox_$(id -un)`.

## Dos trampas que cuestan horas

### `vendorsetup.sh` solo corre con `source build/envsetup.sh`

Si solo relanzas `lunch`, las variables `OF_*` / `FOX_*` **no se refrescan**.
Peor: quitar una variable del fichero no basta, porque si ya estaba exportada
en el shell sobrevive al re-source. Hay que ponerla explicitamente a `0`:

```sh
export FOX_VANILLA_BUILD=0
export FOX_DELETE_INITD_ADDON=0
```

Hay otro detalle relacionado: el script **borra** `/tmp/$DEVICE/fox_env.sh` al
terminar, y ese fichero lo escribe `envsetup.sh` durante `lunch`. Por eso la
primera build funciona y las siguientes salen raras.

### El staging del ramdisk no siempre se reinstala

En builds incrementales, la regla que copia por ejemplo `libminuitwrp.so` al
ramdisk **no vuelve a ejecutarse**: la libreria se re-enlaza pero la imagen
sigue llevando la vieja. Es muy dificil de depurar porque parece que tus
cambios no hacen nada.

Se comprueba comparando md5:

```bash
md5sum $OUT/recovery/root/system/lib64/libminuitwrp.so
md5sum out/soong/.intermediates/bootable/recovery/minuitwrp/libminuitwrp/*/libminuitwrp.so
```

`build.sh` borra el staging antes de compilar para evitarlo.
