# Descifrado FBE

El recovery descifra `/data` con **PIN, patron o contrasena**, y tambien
cuando **no hay ninguna credencial**. Hubo que arreglar tres cosas distintas.

## 1. Keymaster: el de R vendor crashea en S

El device tree publico traia el keymaster de MediaTek de **vendor R**:

```
init: Service 'keymaster-4-0-beanpod' (pid 760) received signal 11
```

SIGSEGV. Y sin keymaster no hay descifrado posible: TWRP se queda esperando
para siempre en la pantalla de contrasena.

La causa es que `libTEECommon.so` **no es compatible entre R y S**. Comparando
md5 contra el `/vendor` real del telefono:

| Fichero | R vendor (arbol) | S vendor (real) |
|---|---|---|
| `libTEECommon.so` | `a6d7385c...` | `b8ca8e87...` |
| servicio keymaster | `@4.0-service.beanpod` | `@4.1-service.beanpod` |

El TEE en si funcionaba perfectamente (`TEEI_BOOT_OK` en dmesg, `/dev/ut_keymaster`
presente); era el binario del HAL el que moria.

**Arreglo**: se empaqueta el `@4.1-service.beanpod` de vendor S junto con
`libkeymaster41.so` y la `libTEECommon.so` correcta.

## 2. El manifiesto VINTF declaraba la version equivocada

Con el binario bueno, el servicio seguia abortando:

```
hwservicemanager: getTransport: Cannot find entry
  android.hardware.keymaster@4.1::IKeymasterDevice/default in VINTF manifest
E HidlServiceManagement: Service ... must be in VINTF manifest in order to register
F android.hardware.keymaster@4.1-service.beanpod: Could not register service (-2147483648)
```

Y sin embargo el log **anterior** mostraba que la conexion con el TEE iba bien:

```
I beanpodkeymaster: keymaster connect start
I beanpodkeymaster: km handle_=6
I beanpodkeymaster: cmd:48 Received 4 byte response
```

El manifiesto del recovery declaraba solo `@4.0`. Ahora declara **las dos**,
asi el mismo arbol vale para una ROM de vendor R y para una de S.

## 3. Sin credencial: dos bugs de memoria sin inicializar

Con PIN funcionaba. Sin ninguna credencial, no.

```
Failed to read '/data/system_de/0/spblob/...pwd'
using secdis to decrypt spblob
spblob v2 / v3
fscrypt_unlock_user_key returned fail
```

El `.pwd` no existe cuando no hay credencial, y eso **no es el problema**: el
codigo sigue por la via `secdis` y llega hasta el final.

### El token rellenado a medias

```c
unsigned char password_token[PASSWORD_TOKEN_SIZE];   // 32 bytes, SIN inicializar
std::string defpassword = "default-password";        // 16 bytes
memcpy(password_token, defpassword.data(), 16);      // los otros 16, basura
```

AOSP, en `SyntheticPasswordManager.stretchLskf()`, hace
`Arrays.copyOf(DEFAULT_PASSWORD, STRETCHED_LSKF_LENGTH)`: `"default-password"`
**rellenado con ceros hasta 32 bytes**. TWRP escribia solo los 16 primeros y
dejaba el resto sin inicializar. Con `-ftrivial-auto-var-init=pattern` esos
bytes valen `0xAA...`, contaminan el `application_id` y la clave derivada sale
mal.

El arreglo es literalmente `= {0}`.

### El tag GCM que nadie comprobaba

Esto es lo que hacia el bug anterior tan dificil de encontrar:

```c
unsigned char tag[AES_BLOCK_SIZE];                      // sin inicializar
EVP_CIPHER_CTX_ctrl(d_ctx, EVP_CTRL_GCM_SET_TAG, 16, tag);
EVP_DecryptFinal_ex(d_ctx, secret_key + actual_size, &final_size);   // resultado ignorado
```

AES-GCM es autenticado: si la clave es incorrecta, el tag no cuadra y hay que
abortar. Aqui se pasaba un buffer sin inicializar como tag y se ignoraba el
resultado, asi que **con clave incorrecta devolvia basura en silencio** y el
fallo solo aparecia mucho despues, en `fscrypt_unlock_user_key`, sin ninguna
pista de la causa.

Ahora se extrae el tag real (ultimos 16 bytes del texto cifrado) y se verifica.
Ademas de ser un bug real, sirvio de **oraculo** para acertar con el
`application_id`: con 80 bytes decia "tag incorrecto", con 96 "tag correcto".

## Comprobar que funciona

```bash
adb shell getprop twrp.user.0.decrypt     # 1 = usuario descifrado
adb shell ls /data/media/0                # nombres legibles, no cifrados
```

Si salen nombres tipo `,UcenBAAAAwSVcMHvVFFzB68AFOFTCRC`, la capa de metadata
esta descifrada pero **el usuario no**.
