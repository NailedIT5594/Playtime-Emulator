# Playtime Emulator

A native Windows desktop app with a Steam-client-inspired interface. It scans
your installed Steam games and can keep a game "running" locally so Steam
accrues playtime — without Python, a terminal, a browser, or a `start.bat`.

> Use at your own risk. Modifying game files and faking playtime may violate
> Steam's terms and can affect refund eligibility. Anti-cheat protected games
> are detected and require explicit confirmation.

## Daily use

Build (or download) one of the two Windows artifacts:

- **Installer:** `dist/Playtime-Emulator-Setup-<version>-x64.exe`
- **Portable:** `dist/Playtime-Emulator-<version>-portable.exe`

Double-click the `.exe`. The app opens in its own native window; no browser,
PowerShell window, or batch file is involved.

1. Pick a game from the library.
2. Choose the executable to replace (the app suggests the best match).
3. Press **START EMULATOR**. If "Launch through Steam" is enabled, Steam is
   asked to start the game immediately.
4. Press **STOP EMULATOR** to terminate the idle process and restore the
   original executable.

## What happens on disk

For the selected executable (`Game.exe`):

```
Game.exe          -> renamed to Game.exe.fakebak
Game.exe          -> replaced by a tiny windowless idle process
Game.exe.fakebak  -> renamed back when you press Stop
```

A `.fakeplay.json` file next to the executable stores the start timestamp.
When a session is stopped, the idle process is terminated and the original
executable is restored automatically. Close the real game before starting;
Windows will not allow a running executable to be renamed.

The idle process is `assets/idle-stub.exe`, a 3.5 KB GUI-subsystem .NET
executable that sleeps with no window. If it is unavailable, the app falls
back to a hidden copy of `cmd.exe` (still terminated automatically on Stop).

## Settings

Settings are saved as `settings.json`:

- Installed build: `%APPDATA%\Playtime Emulator\settings.json`
- Portable build: next to the portable `.exe`
- Extra library folders can be added in the Settings dialog.

The dialog applies appearance changes live and restores the previous look on
Cancel. Available preferences:

- **Paths** — Steam folder and extra library folders.
- **Execution** — launch through Steam by default, telemetry refresh rate.
- **Appearance** — three color themes (Steam, Midnight blue, pure-black OLED),
  five accent presets plus a fully custom color picker, compact/standard/large
  cover sizes, and an offline-friendly cover-art toggle.
- **Accessibility** — interface scale from 90% to 130%, high-contrast mode,
  reduce-motion mode, and always-visible game titles.
- **Library** — pin active sessions to the top of the grid or list.

Interface scale uses Chromium's native page zoom through a minimal, isolated
preload bridge, so the layout never clips at larger sizes.

Steam is detected from the registry and common install locations, then all
`appmanifest_*.acf` files (including `libraryfolders.vdf` libraries) are read
to build the game list. Duplicate app IDs from overlapping library manifests
are ignored.

## Development

Requirements: Node.js 18+ and Windows.

```powershell
npm install
npm start
```

The frontend is bundled and served from a local loopback server inside the
Electron window. It never depends on an external browser.

Useful scripts:

| Script | Purpose |
| --- | --- |
| `npm start` | Run the app in development |
| `master_update.bat` | Build, version, deploy, clean up and relaunch everything in one step |
| `run_update.bat` | Copy frontend changes into the running app without rebuilding |
| `npm run update` | Same as `master_update.bat` |
| `npm run assets` | Regenerate the idle stub and icon |
| `npm run pack` | Unpacked app in `dist/win-unpacked` |
| `npm run dist` | Build the NSIS installer and portable `.exe` |
| `npm run dist:installer` | Build only the installer |
| `npm run dist:portable` | Build only the portable executable |

### Frontend hot updates

`run_update.bat` applies changes to `index.html` (and optional external assets
in a `ui\` folder) without rebuilding the installer or reinstalling the app.
Edit the files in this project, then run:

```bat
run_update.bat
```

It copies the current `index.html` and `ui\` contents to:

- `%APPDATA%\Playtime Emulator\ui` — used by installed and portable builds
- `dist\ui` — portable fallback
- `dist\win-unpacked\ui` — unpacked test build

A running app watches its `ui` folder and reloads automatically. No browser or
external server is involved. The first build that includes this loader must be
installed once; after that, frontend-only updates are instant.

### One-command master update

After changing anything — frontend, backend, assets, build settings, or the
version — run a single command from the project folder:

```bat
master_update.bat
```

That one command:

1. Fingerprints the source and bumps the patch version when something changed.
2. Builds a fresh installer and portable exe when needed (`npm run dist`).
3. Closes a running Playtime Emulator instance just before deploying.
4. Syncs the frontend override with `run_update.bat --no-pause`.
5. Copies `dist\win-unpacked` over the installed folder the Start Menu
   shortcut already points to.
6. Deletes obsolete files left in the installed folder.
7. Repairs the Start Menu and desktop shortcuts and any pinned taskbar shortcut.
8. Updates the Apps & Features name, version and icon.
9. Deletes old `dist\Playtime-Emulator-*` installers for previous versions.
10. Records the deployed fingerprint in `.deploy-state.json` and relaunches the
    new build from the installed folder.

`setup.exe` is never required, and the profile in
`%APPDATA%\Playtime Emulator` is preserved. The installed executable path never
changes, so Windows Search, Start Menu pins and desktop shortcuts keep working.

Useful switches:

| Switch | Effect |
| --- | --- |
| `-NoLaunch` | Deploy without starting the app |
| `-Fast` | Build `dist\win-unpacked` only (`npm run pack`); no installer |
| `-SkipBuild` | Deploy the existing build output without rebuilding or bumping |
| `-NoBump` | Keep the current version even when source changed |
| `-Bump` | Force a patch version bump |
| `-NoClean` | Keep obsolete installed files and old artifacts |
| `-Force` | Rebuild even when the fingerprint says nothing changed (keeps the version) |

For example:

```bat
master_update.bat -NoLaunch
master_update.bat -Fast -NoLaunch
master_update.bat -SkipBuild -NoLaunch
```

Do not launch `dist\win-unpacked\Playtime Emulator.exe` directly for daily use —
that creates a second copy and the Start Menu keeps opening the installed one.

The committed `assets/idle-stub.exe` and `build/icon.png` mean end users and
normal builds do not need a C# compiler. `npm run make-stub` regenerates the
stub with the .NET Framework compiler if you want to change it.

## API

The local server binds to `127.0.0.1` on the first free port starting at
`38472`. All responses are JSON except the UI assets.

| Method | Route | Description |
| --- | --- | --- |
| GET | `/api/settings` | Read persisted settings |
| POST | `/api/settings` | Validate and save settings |
| GET | `/api/meta` | Version, Steam path, platform capabilities |
| GET | `/api/games` | Installed games (`/api/library` is an alias) |
| GET | `/api/sessions` | Active emulation sessions |
| GET | `/api/exes?appid=…` | Executables in a game folder |
| POST | `/api/start` | Swap an executable and optionally launch Steam |
| POST | `/api/stop` | Kill the idle process and restore the executable |

POST requests require the per-launch `X-PE-Token` header injected into the
served HTML. The server also validates the `Host` and `Origin` headers so a
malicious local page cannot drive the API.

## Packaging notes

`electron-builder` produces x64 Windows artifacts under `dist/`. The app is
packed into an ASAR archive with `main.js`, `preload.js`, `index.html`, and
`assets/`; the Node and browser toolchains are not required on the target
machine.

If Windows Defender flags the generated idle stub, add an exclusion or use
the cmd.exe fallback by removing `assets/idle-stub.exe` before building. The
app is unsigned, so SmartScreen may show a "more info" prompt on first run.
