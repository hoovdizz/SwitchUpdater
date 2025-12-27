# Nintendo Switch SD Card Update Script (Version 5)

## Overview

This PowerShell script is an all-in-one maintenance and preparation tool for Nintendo Switch SD cards. It automates backups, crash report cleanup, firmware updates, and the latest payloads for Hekate and Atmosphere.

---

## Features

- **Optional SD Card Backup**
  - Excludes `emuMMC` and `Firmware` folders by default.
  - Creates a timestamped ZIP backup.
- **Crash Report Cleanup**
  - Deletes folders:
    - `/atmosphere/crash_reports`
    - `/atmosphere/erpt_reports`
    - `/atmosphere/fatal_reports`
- **fusee.bin Update**
  - Checks if `fusee.bin` is missing or outdated.
  - Optionally downloads the latest release from [Atmosphere-NX](https://github.com/Atmosphere-NX/Atmosphere/releases/latest).
  - Updates hosts files in `/atmosphere/hosts`:
    - `default.txt`
    - `emummc.txt`
    - `sysmmc.txt`
- **Atmosphere Update**
  - Automatically updates the full Atmosphere release if fusee.bin is updated.
- **Hekate Bootloader & hekate.bin Update**
  - Checks local hekate.bin against the latest release.
  - Optionally updates the full bootloader folder and renames the release bin to `hekate.bin`.
- **Firmware Update**
  - Optionally downloads the latest firmware from [NX_Firmware](https://github.com/THZoria/NX_Firmware/releases) and extracts it to `/Firmware`.
- **GUI Prompts**
  - Uses Windows Forms for yes/no confirmations and file selection dialogs.

---

## Requirements

- **Windows OS**
- **PowerShell 5+**
- **Internet Connection** (for downloading updates and firmware)
- Switch must be in **Hekate USB mode** to detect the SD card.

---

## Usage

1. Connect your Nintendo Switch in **USB mode using Hekate**.
2. Run the script in **PowerShell**:

   ```powershell
   .\SwitchUpdate.ps1
