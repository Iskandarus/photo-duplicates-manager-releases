# Photo Duplicates Manager — Releases

Installers and auto-update feed for **Photo Duplicates Manager**. The source code lives in a private repository; this repo only hosts release assets.

## Downloads

Grab the latest release from the [Releases page](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/latest):

| Platform | File |
| --- | --- |
| Windows x64 | `PDM_<version>_x64-setup.exe` (or `.msi`) |
| macOS Apple Silicon | `PDM_<version>_aarch64.dmg` |
| macOS Intel | `PDM_<version>_x64.dmg` |
| Linux x64 (Debian/Ubuntu) | `PDM_<version>_amd64.deb` |

Installers are currently unsigned — expect a SmartScreen / Gatekeeper warning on first launch.

## Auto-update

The app checks this repo for updates on startup. `latest.json` and the `.app.tar.gz` / `.exe` + `.sig` assets are consumed by the in-app updater (Windows/macOS install in one click; Linux shows a notification pointing here). Do not delete these assets from releases.
