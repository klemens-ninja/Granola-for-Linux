# Granola on Linux

[Granola](https://www.granola.ai) only ships for macOS and Windows. It is an Electron app though, so the macOS `.dmg` already holds all of the app's JavaScript, which runs anywhere. It is just attached to a macOS runtime. This repo swaps in the Linux runtime and fixes what breaks. You get a real Linux app. No Wine, no VM, no emulation.

![Granola running on Linux](docs/screenshot.png)

## Run it

1. Download the **full** `.dmg` from [notes.granola.ai/download?full_dmg=true](https://notes.granola.ai/download?full_dmg=true). The default download on granola.ai is now a ~720 KB stub installer, which does not work here. The right file is roughly 270 MB.
2. Install what you need: `sudo apt install g++-11 nodejs npm python3 curl make patch`
3. Run `./granola-linux.sh "Granola - AI Notepad.dmg"`
4. Open Granola from your app menu and sign in

That is it. The script installs to `~/Applications/granola`, adds a desktop entry, registers the `granola://` sign-in handler, and tests the build before it tells you it worked. Run `./uninstall.sh` to undo it.

Set `INSTALL_DIR=` to install somewhere else. You need x86-64 and g++ 11 or newer. Tested on Pop!\_OS (Ubuntu 20.04 base, glibc 2.32) with Granola 7.452.1 and Electron 42.7.0, and on Debian with g++ 15, Granola 7.749.0 and Electron 45.2.1.

## What works

| | |
|---|---|
| ✅ | Notes, editor, sync, AI features, and search. The core app. |
| ✅ | Sign-in with Google, Microsoft, or SSO |
| ✅ | Encrypted local database that survives restarts |
| ✅ | Microphone recording |
| ⚠️ | System audio capture is limited. The macOS build uses Core Audio to hear the other side of a call. On Linux the app falls back to a browser style capture path. |
| ❌ | Apple Calendar (EventKit). Google and Microsoft calendars still work, since those run on the server. |
| ❌ | Global hotkeys |
| ❌ | Auto-update. Run the script again with a newer `.dmg`. |

## Build it yourself

If you would rather not run the script, the conversion takes six steps (the script does all of them):

1. Extract the `.dmg` with a modern `7zz`. The `p7zip` in most distros cannot read its LZFSE compression.
2. Read the Electron version out of `Electron Framework.framework/.../Info.plist`, then download that exact Linux build.
3. Unzip the Linux runtime into your install folder and delete `resources/default_app.asar`.
4. Copy `app.asar`, `app.asar.unpacked`, and `icons/` from the bundle into the runtime's `resources/`. Skip every Mac binary, since they all sit behind `darwin` checks.
5. Patch the platform string inside `app.asar` so it reports `Windows`. Granola's API returns a 500 error for `platform=linux`, so sign-in fails without this.
6. Rebuild `better-sqlite3-multiple-ciphers` using g++ 11 or newer. Granola's version adds an `updateHook()` that no public build has. Older bundles ship the C++ source in `app.asar.unpacked`. Since 7.749 they do not, so the script builds upstream from npm and applies `patches/better-sqlite3-updatehook.patch`, which re-implements `updateHook(fn)` on top of `sqlite3_update_hook` (callback `(operation, db, table, rowid)`) enables `SQLITE_ENABLE_BYTECODE_VTAB` (the app queries `tables_used()`; without it the UI dies with "Something went wrong"), and adds V8 15 API compatibility for Electron 45.

Four of those six fail with errors that do not point at the real cause. `granola-linux.sh` has the exact commands, with comments explaining each one.

## Troubleshooting

- **`could not read the .dmg` / the `.dmg` is only ~720 KB:** you downloaded the stub installer. Use the full `.dmg` link from step 1.
- **Window shows "Something went wrong" on start:** the native SQLite module is missing a feature. Re-run the script, its smoke test checks `updateHook` and `tables_used()`. To see the real error, start `~/Applications/granola/granola.sh --enable-logging=stderr` from a terminal and look for `react-error` and the line before it.
- **`updateHook patch does not apply`:** Granola moved to a different `better-sqlite3-multiple-ciphers` version. The patch in `patches/` needs a rebase onto that version.
- **Granola starts a second time and nothing happens:** only one instance can run. Quit the old one first.

## Notes

- Nothing here gets around licensing or sign-in. You use your own account and the app talks to Granola's real servers. The only patch is a platform label that their API refuses to accept.
- Granola's code belongs to Granola. Do not commit `app.asar` or the `.dmg`. The `.gitignore` covers both.
- Running the script again is safe. It wipes and rebuilds the install folder and leaves your notes in `~/.config/Granola` alone.
- `./uninstall.sh --purge` also removes the local notes cache and login.
