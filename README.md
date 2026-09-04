# Photo Duplicates Manager

**Photo Duplicates Manager (PDM)** is a cross-platform desktop app that finds and removes duplicate and near-duplicate photos on Windows, macOS, and Linux. Free your disk space and clean up your photo library.

> This repository hosts the **public release assets** (installers, the auto-update manifest, and the resource manager SDK) for Photo Duplicates Manager. The application source code lives in a separate private repository.

## Download

Grab the latest installer from the [**Releases page**](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/latest):

| Platform | File |
| --- | --- |
| Windows x64 | `PDM_<version>_x64-setup.exe` (or `.msi`) |
| macOS Apple Silicon | `PDM_<version>_aarch64.dmg` |
| macOS Intel | `PDM_<version>_x64.dmg` |
| Linux x64 (Debian/Ubuntu) | `PDM_<version>_amd64.deb` |

Installers are currently unsigned - expect a SmartScreen / Gatekeeper warning on first launch.

## Features

- Find exact duplicate photos by content hash
- Detect visually similar (near-duplicate) images
- Preview and compare duplicates side by side before deleting
- Reclaim disk space by removing redundant copies
- Works fully offline - your photos never leave your computer
- Cross-platform: Windows, macOS (Intel + Apple Silicon), and Linux

## Which version goes with which

Two different things are released here and they carry different numbers, because they are numbered by different things. The **application** version is its release tag. The **resource manager SDK** version is the version of the API a resource manager talks to, plus a counter of how many times the same API has been published. Neither can be worked out from the other, so here is what goes with what:

| Application | Core API | Resource manager SDK | Date |
| --- | --- | --- | --- |
| [0.3.0](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/tag/v0.3.0-cm) | 3.0 | [3.0.4](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/tag/sdk-v3.0.4) - the contract, the documents and `pdm-harness` | 2026-09-04 |
| 0.2.2 and earlier | not published | not published | up to 2026-07-28 |

Every application release repeats its own row in its release notes, so the version table travels with the download rather than only living here.

## Writing a resource manager

A **resource manager** is a program that owns a collection of photographs - a folder on a disk, a cloud drive, a photo service - and wants PDM to find the duplicates in it. Anybody can write one; you implement no API and serve nothing to PDM.

Everything you need is one download: **[Resource manager SDK 3.0.4](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/tag/sdk-v3.0.4)**, which is the newest `sdk-v*` release. This line names the current one; the table above says which application version it goes with.

| | |
| --- | --- |
| `pdm-resource-manager-sdk-<version>.zip` | The core API contract (`core-api.v1.yaml`), its changelog, and the documents written for somebody implementing it. Start with `writing-a-resource-manager.md` inside. |
| `pdm-harness-<version>-<platform>` | `pdm-harness` plays the core on a loopback port so you can drive your manager against something that answers, and tells you what it got wrong. One self-contained file - nothing to install. On macOS and Linux, unpack the `.tar.gz` and `chmod +x pdm-harness`. |

Both carry the same number, so a tool and a guide out of one release always describe the same contract.

## Auto-update

The app checks this repository for updates on startup. The `latest.json` manifest and the `.app.tar.gz` / `.exe` + `.sig` assets are consumed by the in-app updater (Windows/macOS install in one click; Linux shows a notification pointing here). **Do not delete these assets from releases.**

An `sdk-v*` release is never the "latest" release, on purpose: `releases/latest` is where every installed application looks for its update manifest, so that place belongs to the application.

## Links

- [Download the latest release](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases/latest)
- [Download page](https://iskandarus.github.io/photo-duplicates-manager-releases/) (GitHub Pages)
- [All releases, including the SDK](https://github.com/Iskandarus/photo-duplicates-manager-releases/releases)

---

_Keywords: photo duplicate finder, duplicate photo remover, find duplicate images, duplicate photo cleaner, remove duplicate pictures, Windows, macOS, Linux._
