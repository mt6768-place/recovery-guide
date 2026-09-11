# FBE decryption

The recovery decrypts `/data` with **PIN, pattern or password**, and also when
there is **no credential at all**. Three separate things had to be fixed.

## 1. The R-vendor keymaster crashes on S

The public device tree shipped MediaTek's keymaster from an **R vendor**:

```
init: Service 'keymaster-4-0-beanpod' (pid 760) received signal 11
```

SIGSEGV. Without a keymaster there is no decryption at all: TWRP waits forever
on the password screen.

The cause is that `libTEECommon.so` is **not ABI compatible between R and S**.
Comparing md5 against the phone's real `/vendor`:

| File | R vendor (tree) | S vendor (real) |
|---|---|---|
| `libTEECommon.so` | `a6d7385c...` | `b8ca8e87...` |
| keymaster service | `@4.0-service.beanpod` | `@4.1-service.beanpod` |

The TEE itself was fine (`TEEI_BOOT_OK` in dmesg, `/dev/ut_keymaster` present);
it was the HAL binary that died.

**Fix**: ship the S-vendor `@4.1-service.beanpod` together with
`libkeymaster41.so` and the matching `libTEECommon.so`.

## 2. The VINTF manifest declared the wrong version

With the right binary the service still aborted:

```
hwservicemanager: getTransport: Cannot find entry
  android.hardware.keymaster@4.1::IKeymasterDevice/default in VINTF manifest
E HidlServiceManagement: Service ... must be in VINTF manifest in order to register
F android.hardware.keymaster@4.1-service.beanpod: Could not register service (-2147483648)
```

Yet the lines just above showed the TEE connection working perfectly:

```
I beanpodkeymaster: keymaster connect start
I beanpodkeymaster: km handle_=6
I beanpodkeymaster: cmd:48 Received 4 byte response
```

The recovery manifest only declared `@4.0`. It now declares **both**, so the
same tree works against an R-vendor ROM and an S-vendor one.

## 3. No credential: two uninitialised-memory bugs

With a PIN it worked. With no credential at all it did not.

```
Failed to read '/data/system_de/0/spblob/...pwd'
using secdis to decrypt spblob
spblob v2 / v3
fscrypt_unlock_user_key returned fail
```

The `.pwd` file does not exist when there is no credential, and that is **not**
the problem: the code falls through to the `secdis` path and gets to the end.

### The half-filled token

```c
unsigned char password_token[PASSWORD_TOKEN_SIZE];   // 32 bytes, uninitialised
std::string defpassword = "default-password";        // 16 bytes
memcpy(password_token, defpassword.data(), 16);      // the other 16 are garbage
```

AOSP, in `SyntheticPasswordManager.stretchLskf()`, does
`Arrays.copyOf(DEFAULT_PASSWORD, STRETCHED_LSKF_LENGTH)`: `"default-password"`
**zero padded to 32 bytes**. TWRP wrote only the first 16 and left the rest
uninitialised. With `-ftrivial-auto-var-init=pattern` those bytes hold
`0xAA...`, they corrupt the `application_id` and the derived key comes out
wrong.

The fix is literally `= {0}`.

### The GCM tag nobody checked

This is what made the bug above so hard to find:

```c
unsigned char tag[AES_BLOCK_SIZE];                      // uninitialised
EVP_CIPHER_CTX_ctrl(d_ctx, EVP_CTRL_GCM_SET_TAG, 16, tag);
EVP_DecryptFinal_ex(d_ctx, secret_key + actual_size, &final_size);   // result ignored
```

AES-GCM is authenticated: if the key is wrong the tag does not match and you
must abort. Here an uninitialised buffer was passed as the tag and the result
was ignored, so a **wrong key returned garbage silently** and the failure only
surfaced much later, in `fscrypt_unlock_user_key`, with no hint of the cause.

The real tag (last 16 bytes of the ciphertext) is now extracted and verified.
Besides being a real bug, it served as an **oracle** to get the
`application_id` right: at 80 bytes it reported "tag mismatch", at 96 "tag ok".

## Verifying

```bash
adb shell getprop twrp.user.0.decrypt     # 1 = user decrypted
adb shell ls /data/media/0                # readable names, not ciphertext
```

Names like `,UcenBAAAAwSVcMHvVFFzB68AFOFTCRC` mean the metadata layer is
decrypted but **the user is not**.
