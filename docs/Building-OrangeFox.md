# Building OrangeFox

```bash
git clone https://github.com/mt6768-place/mt6768-place-guide -b recovery guide
bash guide/scripts/sync.sh ~/fox_12.1
bash guide/scripts/apply-patches.sh ~/fox_12.1
bash guide/scripts/build.sh ~/fox_12.1
```

The image and the flashable zip land in `out/target/product/merlinx/`.

## The four patches

They live in repositories this organisation does not mirror. Without them the
build either fails or the recovery boots but is useless. The first two are
genuine **uninitialised-memory bugs**, only visible because Android builds with
`-ftrivial-auto-var-init=pattern`, which fills locals with `0xAA...`.

### `system/tools/aidl` — the build died at 99%

```cpp
struct ConstReferenceFinder : AidlVisitor {
  const AidlConstantReference* found;   // uninitialised
```

It holds `0xaaaaaaaaaaaaaaaa`, so `if (!found)` never fires, `Find()` returns a
garbage pointer and `AIDL_ERROR()` dereferences it. SIGSEGV on **every AIDL
annotation with parameters**. Fixed with `= nullptr`.

It was found by running the compiled `aidl` under gdb: the crash reproduced
with a single `.aidl` file containing one parameterised annotation.

### `bootable/recovery` — empty `twres/`

The theme is copied during **Soong analysis**, not by a ninja rule, from
`gui/libguitwrp_defaults.go`:

```go
outDir := ctx.Config().Getenv("OUT")
twRes := outDir + "/recovery/root/twres/"
os.MkdirAll(twRes, os.ModePerm)      // error ignored
copyDir(dirToCopy, destDir)          // error ignored
```

If `OUT` arrives as a **relative** path and `soong_build` runs from a different
directory, the destination resolves wrong, `MkdirAll` fails and **every error
is discarded**. The build carries on and dies at 99%:

```
sed: .../recovery/root/twres/splash.xml: No such file or directory
```

The patch anchors `OUT` to an absolute path.

### `system/vold` — decryption

See **[FBE decryption](FBE-decryption.md)**.

### `vendor/recovery` — builds clobbering each other

`OrangeFox_A12.sh` keeps its state in fixed `/tmp` paths
(`/tmp/fox_build_000tmp.txt`, `/tmp/Fox_000_tmp`, `/tmp/oFox00.tmp`...). On a
shared machine, two users building OrangeFox at the same time **read each
other's variables**.

Observed symptom: the image was written to `/OrangeFox-...img`, with `$OUT`
empty, because the script loaded another user's state file for a different
device. The patch moves everything under `/tmp/ofox_$(id -un)`.

## Two traps that cost hours

### `vendorsetup.sh` only runs on `source build/envsetup.sh`

Re-running `lunch` alone does **not** refresh the `OF_*` / `FOX_*` variables.
Worse: removing a variable from the file is not enough, because if it was
already exported in the build shell it survives the re-source. Set it to `0`
explicitly:

```sh
export FOX_VANILLA_BUILD=0
export FOX_DELETE_INITD_ADDON=0
```

There is a related detail: the script **deletes** `/tmp/$DEVICE/fox_env.sh`
when it finishes, and that file is written by `envsetup.sh` during `lunch`.
That is why the first build works and later ones behave oddly.

### The ramdisk staging directory is not always reinstalled

On incremental builds the rule that copies, say, `libminuitwrp.so` into the
ramdisk does **not** re-run: the library is relinked but the image still ships
the old one. It is very hard to debug because it looks like your changes do
nothing.

Check it by comparing md5:

```bash
md5sum $OUT/recovery/root/system/lib64/libminuitwrp.so
md5sum out/soong/.intermediates/bootable/recovery/minuitwrp/libminuitwrp/*/libminuitwrp.so
```

`build.sh` clears the staging directory first to avoid it.
