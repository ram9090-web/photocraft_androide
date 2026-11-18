# PhotoCraft for Android

This adds an Android shell around the existing egui app. **It has not been compiled or run**:
it was written without a Rust toolchain or Android SDK, so expect to fix a few things on the first build
(see "Known risks").

## What was added

| Path | What it is |
|---|---|
| `apps/photocraft-android/` | Rust `cdylib` (`libphotocraft_android.so`). Same `PhotocraftApp` as desktop/web, started through eframe's `android-activity` backend. Modeled on `apps/photocraft-web` (thin host + `Services`). |
| `android/` | Gradle/Kotlin project. `MainActivity` extends `NativeActivity` and provides the system file picker (SAF), the share sheet for saved files, and "Open with"/Share intents, over JNI. |

The workspace already globs `apps/*`, so the new crate is a member automatically (it is not in `default-members`).

## Prerequisites

- Rust 1.95+ (`rust-version` in `Cargo.toml`) and the Android targets:
  `rustup target add aarch64-linux-android x86_64-linux-android`
- `cargo install cargo-ndk`
- Android Studio (or command-line SDK), SDK Platform 34, NDK 26.3.11579264, JDK 17

## Build

```bash
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/26.3.11579264
cd android
gradle wrapper            # once; generates gradlew (no wrapper jar is included)
./gradlew assembleDebug   # runs `cargo ndk ... build --release -p photocraft-android` first
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Or open `android/` in Android Studio and press Run. To build only the native library:
`cargo ndk -t arm64-v8a build --release -p photocraft-android`.

Release builds use the workspace's `lto = "thin"`, `codegen-units = 1`, so expect a long first compile.

## How it behaves

- File › Open: Android's document picker. Chosen files arrive as bytes, not paths.
- File › Save / Export: written to the app's private `exports/` folder, then the share sheet opens so you can move the file to Drive, Files, etc.
- Tapping an image in another app → "Open with PhotoCraft" opens it.
- Preferences persist in `<app files dir>/preferences.json`.
- Not wired on Android: folder pickers, the control/agent server, print, screen capture, self-update, tablet/pen helpers, system font scan.

## Known risks (first things to check)

1. **Desktop-only dependencies may not compile for Android.** `photocraft-ui-egui` enables `craft`'s `dialogs`
   (rfd), `update` and `automation-agents` on every non-wasm target, and craft-libs' own docs list Android as an
   open follow-up for `craft-dialogs`. If `cargo ndk build` fails there, the fix is in
   `storytold/craft-libs` / `crates/ui-egui/Cargo.toml`: gate those features with
   `cfg(not(target_os = "android"))`, the same way wasm is already handled.
2. **Cargo.lock** needs updating (`jni`, `android_logger` are new). Run `cargo update -p photocraft-android --precise`-style
   resolution, or just `cargo check -p photocraft-android --target aarch64-linux-android`.
3. **Workspace lint `unsafe_code = "forbid"`**: the new crate opts out of workspace lints (JNI needs `unsafe`);
   `cargo xtask ci` house rules may flag that and need an exemption.
4. **UI is desktop-shaped.** It runs, but menus, small hit targets and keyboard shortcuts are not touch-optimized;
   a tablet or a phone with a stylus is the realistic target. Touch/stylus feed (`app.stylus.feed`) isn't connected.
5. **Memory.** Large PSDs on a phone will hit the app's own memory gates (`largeHeap` is set).
6. If wgpu picks a bad backend on some device, set `backends` in `android.rs` (Vulkan only, or GL only).

## Licensing

The project is MIT OR Apache-2.0; these additions are under the same terms.
