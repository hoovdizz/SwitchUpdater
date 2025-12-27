<#
.SYNOPSIS
    Nintendo Switch SD Card Maintenance Script

.DESCRIPTION
    This script performs several maintenance tasks on a Nintendo Switch SD card:
    - Optional backup of the SD card
    - Cleanup of crash reports
    - Updates fusee.bin and Atmosphere
    - Updates Hekate bootloader and hekate.bin
    - Downloads latest firmware for installation
    - Pulls updated host files for Atmosphere
    This is an all-in-one solution for Switch SD card preparation and maintenance.

.VERSION
    5.0

.AUTHOR
    Alix Hoover

.DATE
    Created 12/20/2025

.NOTES
    Make sure your Switch is in USB mode using Hekate before running this script.
    Also ensure if you use themes they are uninstalled
#>


Add-Type -AssemblyName System.Windows.Forms

# -----------------------------
# STEP 1: Prompt user to enable Hekate USB mode
# -----------------------------
[System.Windows.Forms.MessageBox]::Show(
    "Please put your Nintendo Switch into USB mode using Hekate.`n`n" +
    "Once the Switch is connected and USB mode is enabled, click OK to continue.",
    "Nintendo Switch USB Mode Required",
    [System.Windows.Forms.MessageBoxButtons]::OK,
    [System.Windows.Forms.MessageBoxIcon]::Information
)

# -----------------------------
# STEP 2: Detect SWITCH SD
# -----------------------------
$volume = Get-Volume | Where-Object FileSystemLabel -eq "SWITCH SD"

if ($null -eq $volume) {
    [System.Windows.Forms.MessageBox]::Show(
        "Switch SD card was NOT detected.`n`n" +
        "Make sure Hekate USB mode is enabled and the Switch is connected.",
        "Device Not Found",
        [System.Windows.Forms.MessageBoxButtons]::OK,
        [System.Windows.Forms.MessageBoxIcon]::Error
    )
    exit 1
}

$driveLetter = $volume.DriveLetter
$basePath    = "$($driveLetter):\"
$didBackup   = $false

# -----------------------------
# STEP 3: Backup confirmation
# -----------------------------
$backupConfirm = [System.Windows.Forms.MessageBox]::Show(
    "Switch SD card detected!`n`n" +
    "Drive Letter: $($driveLetter):`n`n" +
    "A backup of the SD card is about to begin.`n`n" +
    "This step is HIGHLY RECOMMENDED before making any changes.`n`n" +
    "All contents will be archived EXCEPT:`n" +
    "- emuMMC`n" +
    "- Firmware`n`n" +
    "Do you want to continue with the backup?",
    "Switch SD Card Backup (Highly Recommended)",
    [System.Windows.Forms.MessageBoxButtons]::YesNo,
    [System.Windows.Forms.MessageBoxIcon]::Warning
)

# -----------------------------
# STEP 4: Perform backup if approved
# -----------------------------
if ($backupConfirm -eq [System.Windows.Forms.DialogResult]::Yes) {

    $timestamp = Get-Date -Format "yyyy-MM-dd_HHmmss"
    $defaultFileName = "SWITCH_SD_CARD_BACKUP_$timestamp.zip"

    $saveDialog = New-Object System.Windows.Forms.SaveFileDialog
    $saveDialog.Filter = "ZIP Files (*.zip)|*.zip"
    $saveDialog.FileName = $defaultFileName
    $saveDialog.Title = "Save Switch SD Card Backup"

    if ($saveDialog.ShowDialog() -eq [System.Windows.Forms.DialogResult]::OK) {

        $zipPath = $saveDialog.FileName

        $itemsToBackup = Get-ChildItem -Path $basePath -Force |
            Where-Object { $_.Name -notin @("emuMMC", "Firmware") }

        Compress-Archive `
            -Path $itemsToBackup.FullName `
            -DestinationPath $zipPath `
            -Force

        $didBackup = $true
    }
    else {
        [System.Windows.Forms.MessageBox]::Show(
            "Backup cancelled. Cleanup will still continue.",
            "Backup Skipped",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Information
        )
    }
}

# -----------------------------
# FUNCTION: Compute SHA-256 Hash
# -----------------------------
function Get-FileHashSHA256($path) {
    if (-Not (Test-Path $path)) { return $null }
    return (Get-FileHash -Path $path -Algorithm SHA256).Hash.ToLower()
}

# -----------------------------
# STEP 5: fusee.bin check/update + Atmosphere full update + hosts files
# -----------------------------
$localFuseePath = Join-Path $basePath "bootloader\payloads\fusee.bin"
$fuseeStatus = "⚠ fusee.bin check skipped"

# Download latest fusee.bin from Atmosphere release
$repoOwner = "Atmosphere-NX"
$repoName  = "Atmosphere"
$apiUrl    = "https://api.github.com/repos/$repoOwner/$repoName/releases/latest"

try { $release = Invoke-RestMethod -Uri $apiUrl -UseBasicParsing } catch { $release = $null }

if ($release) {
    $latestTag = $release.tag_name
    $fuseeAsset = $release.assets | Where-Object { $_.name -match "fusee.*\.bin$" } | Select-Object -First 1

    if ($fuseeAsset) {
        $fuseeUrl = $fuseeAsset.browser_download_url
        $tempFusee = Join-Path $env:TEMP "latest_fusee.bin"
        Invoke-WebRequest -Uri $fuseeUrl -OutFile $tempFusee -UseBasicParsing

        $latestHash = Get-FileHashSHA256 $tempFusee
        $localHash  = Get-FileHashSHA256 $localFuseePath

        if (-Not $localHash -or $localHash -ne $latestHash) {
            # fusee.bin missing or outdated → prompt user
            $userChoice = [System.Windows.Forms.MessageBox]::Show(
                "fusee.bin is missing or out-of-date.`n`n" +
                "Latest release: $latestTag`n`n" +
                "Do you want to download the latest fusee.bin, update Atmosphere, and pull hosts files?",
                "Update fusee.bin & Atmosphere & Hosts?",
                [System.Windows.Forms.MessageBoxButtons]::YesNo,
                [System.Windows.Forms.MessageBoxIcon]::Question
            )

            if ($userChoice -eq [System.Windows.Forms.DialogResult]::Yes) {

                # Ensure local fusee folder exists
                $fuseeFolder = Split-Path $localFuseePath
                if (-Not (Test-Path $fuseeFolder)) { New-Item -Path $fuseeFolder -ItemType Directory | Out-Null }

                # Copy fusee.bin
                Copy-Item -Path $tempFusee -Destination $localFuseePath -Force

                # Download latest Atmosphere ZIP
                $zipAsset = $release.assets | Where-Object { $_.name -match "\.zip$" } | Select-Object -First 1
                if ($zipAsset) {
                    $zipUrl = $zipAsset.browser_download_url
                    $tempZip = Join-Path $env:TEMP "latest_atmosphere.zip"
                    Invoke-WebRequest -Uri $zipUrl -OutFile $tempZip -UseBasicParsing

                    # Extract all contents to basePath (overwrite)
                    Expand-Archive -Path $tempZip -DestinationPath $basePath -Force
                    Remove-Item $tempZip -Force

                    # Pull hosts files
                    $hostsPath = Join-Path $basePath "atmosphere\hosts"
                    if (-not (Test-Path $hostsPath)) { New-Item -Path $hostsPath -ItemType Directory | Out-Null }

                    $hostsFiles = @(
                        "https://github.com/hoovdizz/SwitchUpdater/raw/refs/heads/main/default.txt",
                        "https://github.com/hoovdizz/SwitchUpdater/raw/refs/heads/main/emummc.txt",
                        "https://github.com/hoovdizz/SwitchUpdater/raw/refs/heads/main/sysmmc.txt"
                    )

                    foreach ($fileUrl in $hostsFiles) {
                        $fileName = Split-Path $fileUrl -Leaf
                        $destPath = Join-Path $hostsPath $fileName
                        Invoke-WebRequest -Uri $fileUrl -OutFile $destPath -UseBasicParsing
                    }

                    # Verify fusee.bin hash after extraction
                    $newLocalHash = Get-FileHashSHA256 $localFuseePath
                    if ($newLocalHash -eq $latestHash) {
                        $fuseeStatus = "✔ fusee.bin, Atmosphere, and hosts files updated"
                        [System.Windows.Forms.MessageBox]::Show(
                            "fusee.bin, Atmosphere, and hosts files successfully updated and verified.",
                            "Update Complete",
                            [System.Windows.Forms.MessageBoxButtons]::OK,
                            [System.Windows.Forms.MessageBoxIcon]::Information
                        )
                    } else {
                        $fuseeStatus = "✖ fusee.bin update failed after Atmosphere extraction"
                        [System.Windows.Forms.MessageBox]::Show(
                            "Warning: fusee.bin hash mismatch after Atmosphere extraction!",
                            "Update Failed",
                            [System.Windows.Forms.MessageBoxButtons]::OK,
                            [System.Windows.Forms.MessageBoxIcon]::Error
                        )
                    }
                } else {
                    $fuseeStatus = "✖ Atmosphere ZIP asset not found"
                }
            } else {
                $fuseeStatus = "✖ fusee.bin not updated"
            }
        } else {
            $fuseeStatus = "✔ fusee.bin up-to-date"
        }

        Remove-Item $tempFusee -Force
    }
} else {
    $fuseeStatus = "✖ Failed to fetch Atmosphere release info"
}

# -----------------------------
# STEP 6: hekate.bin + bootloader update
# -----------------------------
$localHekatePath = Join-Path $basePath "hekate.bin"
$hekateStatus = "⚠ hekate.bin check skipped"

$localHash = Get-FileHashSHA256 $localHekatePath
$hekateApiUrl = "https://api.github.com/repos/CTCaer/hekate/releases/latest"
try { $hekateRelease = Invoke-RestMethod -Uri $hekateApiUrl -UseBasicParsing } catch { $hekateRelease = $null }

if ($hekateRelease) {
    $latestTag = $hekateRelease.tag_name
    $binAsset = $hekateRelease.assets | Where-Object { $_.name -match "\.bin$" } | Select-Object -First 1

    if ($binAsset) {
        $binUrl = $binAsset.browser_download_url
        $tempHekate = Join-Path $env:TEMP "latest_hekate_temp.bin"
        Invoke-WebRequest -Uri $binUrl -OutFile $tempHekate -UseBasicParsing
        $latestHash = Get-FileHashSHA256 $tempHekate

        if (-not $localHash -or $localHash -ne $latestHash) {
            $userChoice = [System.Windows.Forms.MessageBox]::Show(
                "Local hekate.bin is missing or out-of-date.`n`n" +
                "Latest release: $latestTag`n`n" +
                "Do you want to update the bootloader folder and hekate.bin with the latest version?",
                "Update hekate bootloader & bin?",
                [System.Windows.Forms.MessageBoxButtons]::YesNo,
                [System.Windows.Forms.MessageBoxIcon]::Question
            )

            if ($userChoice -eq [System.Windows.Forms.DialogResult]::Yes) {
                $zipAsset = $hekateRelease.assets | Where-Object { $_.name -match "\.zip$" } | Select-Object -First 1
                if ($zipAsset) {
                    $zipUrl = $zipAsset.browser_download_url
                    $tempZip = Join-Path $env:TEMP "latest_hekate.zip"
                    Invoke-WebRequest -Uri $zipUrl -OutFile $tempZip -UseBasicParsing

                    $tempExtract = Join-Path $env:TEMP "hekate_extract"
                    if (Test-Path $tempExtract) { Remove-Item $tempExtract -Recurse -Force }
                    Expand-Archive -Path $tempZip -DestinationPath $tempExtract -Force

                    $bootloaderSource = Join-Path $tempExtract "bootloader"
                    $bootloaderDest   = Join-Path $basePath "bootloader"

                    if (Test-Path $bootloaderSource) {
                        Copy-Item -Path $bootloaderSource\* -Destination $bootloaderDest -Recurse -Force

                        $zipBinFile = Get-ChildItem -Path $tempExtract -Filter "*.bin" | Select-Object -First 1
                        if ($zipBinFile) {
                            Copy-Item -Path $zipBinFile.FullName -Destination $localHekatePath -Force
                            $hekateStatus = "✔ hekate bootloader and bin updated to $latestTag"
                        } else {
                            $hekateStatus = "✖ hekate.bin not found in ZIP, bootloader updated only"
                        }
                    } else {
                        $hekateStatus = "✖ bootloader folder not found in ZIP"
                    }

                    Remove-Item $tempZip -Force
                    Remove-Item $tempExtract -Recurse -Force
                } else {
                    $hekateStatus = "✖ Hekate ZIP asset not found"
                }
            } else {
                $hekateStatus = "✖ hekate.bin not updated"
            }
        } else {
            $hekateStatus = "✔ hekate.bin up-to-date ($latestTag)"
        }

        Remove-Item $tempHekate -Force
    }
} else {
    $hekateStatus = "✖ Failed to fetch Hekate release info"
}

# -----------------------------
# STEP 7: Crash report cleanup
# -----------------------------
$pathsToDelete = @(
    "atmosphere\crash_reports",
    "atmosphere\erpt_reports",
    "atmosphere\fatal_reports"
)

$pathList = ($pathsToDelete | ForEach-Object { "$basePath$_" }) -join "`n"

$cleanupConfirm = [System.Windows.Forms.MessageBox]::Show(
    "The following crash report folders will now be deleted:`n`n$pathList`n`nDo you want to continue?",
    "Confirm Crash Report Cleanup",
    [System.Windows.Forms.MessageBoxButtons]::YesNo,
    [System.Windows.Forms.MessageBoxIcon]::Warning
)

if ($cleanupConfirm -eq [System.Windows.Forms.DialogResult]::Yes) {
    foreach ($relativePath in $pathsToDelete) {
        $fullPath = Join-Path $basePath $relativePath
        if (Test-Path $fullPath) { Remove-Item -Path $fullPath -Recurse -Force -ErrorAction SilentlyContinue }
    }
} else {
    [System.Windows.Forms.MessageBox]::Show("Crash report cleanup skipped.", "Cleanup Skipped", [System.Windows.Forms.MessageBoxButtons]::OK, [System.Windows.Forms.MessageBoxIcon]::Information)
}

# -----------------------------
# STEP 8: Pull latest firmware
# -----------------------------
$firmwareConfirm = [System.Windows.Forms.MessageBox]::Show(
    "Do you want to pull the latest firmware onto the SD card for installation?",
    "Update Firmware",
    [System.Windows.Forms.MessageBoxButtons]::YesNo,
    [System.Windows.Forms.MessageBoxIcon]::Question
)

if ($firmwareConfirm -eq [System.Windows.Forms.DialogResult]::Yes) {

    $firmwarePath = Join-Path $basePath "Firmware"

    if (Test-Path $firmwarePath) {
        Remove-Item -Path (Join-Path $firmwarePath "*") -Recurse -Force -ErrorAction SilentlyContinue
    } else {
        New-Item -Path $firmwarePath -ItemType Directory | Out-Null
    }

    $firmwareApiUrl = "https://api.github.com/repos/THZoria/NX_Firmware/releases/latest"
    try { $firmwareRelease = Invoke-RestMethod -Uri $firmwareApiUrl -UseBasicParsing } catch { $firmwareRelease = $null }

    if ($firmwareRelease) {
        $zipAsset = $firmwareRelease.assets | Where-Object { $_.name -match "\.zip$" } | Select-Object -First 1
        if ($zipAsset) {
            $zipUrl = $zipAsset.browser_download_url
            $tempZip = Join-Path $env:TEMP "latest_firmware.zip"
            Invoke-WebRequest -Uri $zipUrl -OutFile $tempZip -UseBasicParsing

            Expand-Archive -Path $tempZip -DestinationPath $firmwarePath -Force
            Remove-Item $tempZip -Force

            $firmwareStatus = "✔ Firmware updated"
        } else {
            [System.Windows.Forms.MessageBox]::Show(
                "Could not find a ZIP asset in the latest firmware release.",
                "Firmware Update Failed",
                [System.Windows.Forms.MessageBoxButtons]::OK,
                [System.Windows.Forms.MessageBoxIcon]::Error
            )
            $firmwareStatus = "✖ Firmware update failed"
        }
    } else {
        [System.Windows.Forms.MessageBox]::Show(
            "Failed to fetch latest firmware release information.",
            "Firmware Update Failed",
            [System.Windows.Forms.MessageBoxButtons]::OK,
            [System.Windows.Forms.MessageBoxIcon]::Error
        )
        $firmwareStatus = "✖ Firmware update failed"
    }
} else {
    $firmwareStatus = "✖ Firmware update skipped"
}

# -----------------------------
# STEP 9: Final summary
# -----------------------------
if ($didBackup) {
    $backupStatus = "✔ Backup created"
} else {
    $backupStatus = "✖ Backup skipped"
}

$finalMessage = "Operation completed!`n`n"
$finalMessage += "$backupStatus`n"
$finalMessage += "$fuseeStatus`n"
$finalMessage += "$hekateStatus`n"
$finalMessage += "$firmwareStatus`n"
$finalMessage += "✔ Crash reports removed"

[System.Windows.Forms.MessageBox]::Show(
    $finalMessage,
    "Switch SD Maintenance Summary",
    [System.Windows.Forms.MessageBoxButtons]::OK,
    [System.Windows.Forms.MessageBoxIcon]::Information
)
