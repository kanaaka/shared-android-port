# shared-android-port

The pieces every port in the **Craft-app Android set** shares: the cross-compile environment, two
build/verify helpers, and the three source files that fix the platform gaps all of them hit.

These are the copies the sibling repositories carry under `scripts/` and in their port crates; this
repository is where they are canonical.

| File | What it is |
|---|---|
| `android-env.sh` | the cross-compile environment: rustup's `bin`, the JDK, `ANDROID_HOME`/NDK, and — the part that is easy to miss — the NDK's `bin` on `PATH` plus `CC`/`CXX`/`AR`/`RANLIB` per target, which the C dependencies (`aws-lc-sys` for PdfCraft) need. Also the `x86_64-linux-android` linker, for running a build in an emulator on an x86_64 host. |
| `build-apk.sh` | `cargo apk build --release -p <crate>` with that app's sideload keystore. Usage: `build-apk.sh <repo> <crate> [features…]` |
| `verify-apk.sh` | checks the finished artifact: `aapt2 dump badging`, `apksigner verify`, `llvm-nm` for the two exported symbols (`android_main`, `ANativeActivity_onCreate`), the `.so` inside the APK, the injected `android.app.lib_name`, and the theme attribute resolved by id. |
| `file_picker.rs` | the in-app file picker, app-agnostic: a global request queue plus `draw(ctx)`, readable-root probing, extension filtering as the union of what the app asked for, "Show all files" with a hidden-file count, one/many/save/folder dialogs, and locked roots that name the permission they need. Written for a tablet: sized from `ctx.content_rect()`, enlarged padding, listings cached per visit. |
| `modifiers.rs` | modifier-key tracking for a shell's `raw_input_hook`. winit's Android backend never emits `ModifiersChanged` and `egui-winit` reads modifiers only from that event, so `Ctrl`/`Shift`/`Alt` are otherwise permanently false. |
| `prefs_watch.rs` | preferences written *while the app runs* (change-detected, at most every two seconds, atomic rename), plus the live-window count a quit uses to end one window instead of the process. |

## The platform gaps these fill

Android breaks four things in a Rust/eframe desktop app, all of them silently:

1. **No native file dialogs.** `rfd` compiles no backend for Android; the Storage Access Framework
   needs `startActivityForResult`, which `android-activity` 0.6 does not expose. Hence a picker drawn
   with egui (`file_picker.rs`, or the `crate::rfd` stand-in PdfCraft uses).
2. **No modifier keys.** `modifiers.rs`.
3. **No preferences moment.** The desktop writes its preferences on the way out; Android gives no such
   event, so a swipe from Recents loses them. `prefs_watch.rs`.
4. **`ViewportCommand::Close` closes nothing.** An in-app Exit has to end the process itself, and it
   has to flush the log first because a process exit runs no `on_exit`.

Two more that are worth knowing before writing the shell: `accesskit` and `android-native-activity`
are mutually exclusive in `eframe` (a `compile_error!`), so the UI crate has to gate `accesskit` to
non-Android; and the APK's `android.app.lib_name` is derived from the crate's `[lib] name` with `-` →
`_`, so a rename there yields an APK that installs and dies on launch.

## Usage

```bash
. ./android-env.sh
./build-apk.sh deckcraft deckcraft-android          # repo dir + crate
./verify-apk.sh /path/to/deckcraft-0.5.0-arm64-v8a.apk
```

Paths inside `android-env.sh` and `build-apk.sh` are written for the Windows machine these ports were
built on (`$HOME/projects/<app>`, `C:/Users/Admin/android-sdk`); adjust them for yours.

## Licence

MIT — see [`LICENSE`](LICENSE). The scripts and source files here were written for these ports and
carry no upstream code.
