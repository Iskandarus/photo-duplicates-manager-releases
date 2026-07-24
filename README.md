# Photo Duplicates Manager

**Photo Duplicates Manager (PDM)** is a cross-platform desktop app that finds and removes duplicate and near-duplicate photos on Windows, macOS, and Linux. Free your disk space and clean up your photo library.

> This repository hosts the **public release assets** (installers + auto-update manifest) for Photo Duplicates Manager. The application source code lives in a separate private repository.

## Download

Grab the latest installer from the [**Releases page**](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/latest):

| Platform | File |
| --- | --- |
| Windows x64 | `PDM_<version>_x64-setup.exe` (or `.msi`) |
| macOS Apple Silicon | `PDM_<version>_aarch64.dmg` |
| macOS Intel | `PDM_<version>_x64.dmg` |
| Linux x64 (Debian/Ubuntu) | `PDM_<version>_amd64.deb` |

Installers are currently unsigned — expect a SmartScreen / Gatekeeper warning on first launch.

## Features

- Find exact duplicate photos by content hash
- Detect visually similar (near-duplicate) images
- Preview and compare duplicates side by side before deleting
- Reclaim disk space by removing redundant copies
- Works fully offline — your photos never leave your computer
- Cross-platform: Windows, macOS (Intel + Apple Silicon), and Linux

## Auto-update

The app checks this repository for updates on startup. The `latest.json` manifest and the `.app.tar.gz` / `.exe` + `.sig` assets are consumed by the in-app updater (Windows/macOS install in one click; Linux shows a notification pointing here). **Do not delete these assets from releases.**

## Links

- 📥 [Download the latest release](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/latest)
- 🌐 [Download page](https://iskandarus.github.io/photo-duplicates-manager-releases/) (GitHub Pages)

---

_Keywords: photo duplicate finder, duplicate photo remover, find duplicate images, duplicate photo cleaner, remove duplicate pictures, Windows, macOS, Linux._
