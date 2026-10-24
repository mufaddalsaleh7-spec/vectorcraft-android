# Building the VectorCraft APK (GitHub Actions)

The workflow [`build-apk.yml`](../.github/workflows/build-apk.yml) turns this repository into a
signed, installable arm64-v8a APK. It runs the same assembly the README describes by hand — upstream
`storytold/vectorcraft` at the pinned commit, with `apps/vectorcraft-android/` copied in — and then
cross-compiles and signs it with `cargo apk`.

## Download the current build

| | |
|---|---|
| Release | [`ci-latest`](https://github.com/mufaddalsaleh7-spec/vectorcraft-android/releases/tag/ci-latest) |
| File | `vectorcraft-0.9.0-arm64-v8a-20261024.apk` |
| Size | 31,023,933 bytes (29.6 MiB) |
| sha256 | `efa5d51e32295cd425dd37ddabd9b8b05b105814309af8f841263a2c87ab6889` |
| Package | `ai.storyteller.vectorcraft` — versionName `0.9.0`, versionCode `16779520` |
| ABI | `arm64-v8a` only |
| minSdk / targetSdk | 26 / 35 |
| Signature | v2 + v3, throwaway sideload key generated in CI (not an official key) |

Install it by copying the APK to the device and opening it with *Install unknown apps* allowed, or
with `adb install -r vectorcraft-0.9.0-arm64-v8a-20261024.apk`.

## What the workflow does

| Step | Why |
|---|---|
| JDK 21, Rust stable + `aarch64-linux-android` | `cargo apk` needs both; `apksigner` runs on the JDK |
| SDK packages | `platforms;android-35`, `build-tools;35.0.0`, `ndk;27.2.12479018` — the NDK r27 the port targets |
| NDK toolchain env | `CARGO_TARGET_AARCH64_LINUX_ANDROID_LINKER=aarch64-linux-android26-clang` (26 = minSdk), `CC`/`CXX`/`AR` for any C dependency, and `llvm-strip` placed on `PATH` as `strip` because the shell crate's metadata asks for `strip` by name |
| `cargo install cargo-apk` | v0.10.0; `--locked` first, plain install as a fallback |
| Assemble | `codeload` tarball of the pin → copy in `apps/vectorcraft-android/`; nothing upstream is edited |
| Keystore | `keytool` PKCS#12 key, restored from the Actions cache on later runs so builds install over each other |
| `cargo apk build --release -p vectorcraft-android` | builds `libvectorcraft.so` for `arm64-v8a` and packages it as a NativeActivity APK |
| Verify | `aapt2 dump badging`, `apksigner verify`, the `.so` listing, `llvm-nm` for `android_main` + `ANativeActivity_onCreate`, `sha256sum` |
| Publish | artifact for the run, plus the APK on the `ci-latest` release |

Re-trigger it with any push that touches `apps/vectorcraft-android/**` or the workflow file, or by
hand with *Actions ▸ Build VectorCraft APK ▸ Run workflow*. The runner's elapsed time for the build
recorded below was 10m 48s.

## Verification output of the published build

Taken from the workflow's own "Verify the artifact" step (`run 40123890144`, commit `f756d0c`):

```
package: name='ai.storyteller.vectorcraft' versionCode='16779520' versionName='0.9.0' ... compileSdkVersion='35'
targetSdkVersion:'35'
uses-permission: name='android.permission.MANAGE_EXTERNAL_STORAGE'
uses-permission: name='android.permission.READ_MEDIA_IMAGES'
uses-permission: name='android.permission.READ_EXTERNAL_STORAGE' maxSdkVersion='32'
uses-permission: name='android.permission.WRITE_EXTERNAL_STORAGE' maxSdkVersion='29'
application-label:'VectorCraft'
launchable-activity: name='android.app.NativeActivity'
native-code: 'arm64-v8a'
---- signature
Verifies
Verified using v2 scheme (APK Signature Scheme v2): true
Verified using v3 scheme (APK Signature Scheme v3): true
---- native code inside the APK
lib/arm64-v8a/libvectorcraft.so
---- exported native symbols
000000000131c6e8 T ANativeActivity_onCreate
0000000001dae88c T android_main
---- sha256
efa5d51e32295cd425dd37ddabd9b8b05b105814309af8f841263a2c87ab6889
```

`android_main` and `ANativeActivity_onCreate` being exported is what makes the APK launch rather than
install and die: NativeActivity loads `libvectorcraft.so` and needs the entry point there.

## What this does *not* prove

The build, the manifest, the signature and the native entry points are checked. Nothing here runs the
app: no device or emulator test is part of this workflow, so the behavioural claims still rest on the
tablet testing recorded in [`PORT-NOTES.md`](PORT-NOTES.md). The known gaps — one window, no audio,
preferences written only on a clean exit — are unchanged; the modifier-key fix is included, since it
is part of `apps/vectorcraft-android/`.

## Building it yourself instead

The same steps run on a workstation; see *Build it* in the [README](../README.md). The only things
this workflow adds are the exact tool versions and the CI-generated signing key.
