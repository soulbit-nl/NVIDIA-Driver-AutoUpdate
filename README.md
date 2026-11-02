# NVIDIA-Driver-AutoUpdate

A PowerShell script for Windows admins that keeps NVIDIA GPU drivers current without user interaction. It detects the GPU, looks up the latest driver at NVIDIA, checks the NVIDIA Authenticode signature of the download and installs it silently. It never downgrades and returns `0`, `1` or `3010`, so it fits Microsoft Intune (Win32 app, detection and requirement scripts included), a scheduled task or any other deployment tool.

Maintained by Dennis Biswane ([SoulBit](https://soulbit.nl)).

Based on [jakelmg/NVIDIA-Driver-Downloader](https://github.com/jakelmg/NVIDIA-Driver-Downloader) (MIT), reworked for unattended use.

## Scripts

| Script | Purpose |
| --- | --- |
| [`scripts/Update-NVIDIADriver.ps1`](scripts/Update-NVIDIADriver.ps1) | The main script. Finds the GPU, looks up the latest driver at NVIDIA, downloads, verifies and installs it. |
| [`intune/Detect-NVIDIADriver.ps1`](intune/Detect-NVIDIADriver.ps1) | Optional, Intune example. Detection rule: not detected (exit 1) when a newer driver exists or the lookup fails. |
| [`intune/Requirement-NVIDIAGPU.ps1`](intune/Requirement-NVIDIAGPU.ps1) | Optional, Intune example. Requirement rule: returns the number of NVIDIA GPUs. |

## What the update script does

- Detects the GPU by PCI vendor ID (`10DE`), not by display name.
- Maps the Windows GPU name to NVIDIA's product list (handles "Generation" naming, keeps laptop and desktop variants apart).
- When one GPU name matches several NVIDIA products (for example `Quadro T2000` in both the notebook and the embedded series), it compares the device form factor (SMBIOS chassis type and power profile; known signals must agree) with each candidate's NVIDIA product series. A series counts as notebook or embedded by keyword, and as desktop only by heuristic: when a sibling series with the same name plus "(Notebooks)" exists. If that does not settle it, it continues only when all candidates resolve to the same driver package; otherwise it stops.
- A name ending in "with Max-Q Design" without an exact match is retried without that suffix, but only on a device detected as a notebook and only against notebook series.
- Every mapping step is logged with the raw chassis and power profile values. The detection script uses the same logic.
- Never downgrades and never reinstalls the same version.
- Downloads with resume, retries, mirror failover and a stall timeout (suited for slow links).
- Validates the NVIDIA Authenticode signature before running the package.
- Installs silently (`-s -noreboot`, optional `-clean`) and verifies exit code and resulting driver version.
- Logs to `%ProgramData%\NVIDIA-DriverAutoUpdate\Logs\NVIDIA-Driver.log` (rotates at 5 MB). Use `-IntuneLog` for the Intune Management Extension log folder or `-LogPath` for any folder.
- Exit codes: `0` success/compliant, `1` error, `3010` success, reboot required.

### Parameters

| Parameter | Description |
| --- | --- |
| `-CheckOnly` | Only report the current and the latest available version. Nothing is downloaded or installed. The script does not require administrator rights for this mode. |
| `-Clean` | Adds NVIDIA's `-clean` switch, so the installer does a clean installation (the same as ticking "Perform a clean installation" in the setup): previous driver files and NVIDIA settings are removed and reset to defaults. **Without `-Clean` the script runs a normal silent upgrade** (`-s -noreboot`) that keeps your existing settings. Only used when an update is actually needed. |
| `-IntuneLog` | Log to the Intune Management Extension log folder instead of the default. |
| `-LogPath <folder>` | Log to a custom folder. |
| `-ProductSeriesId <n>` and `-ProductId <n>` | NVIDIA's numeric IDs for your GPU (`psid` and `pfid`). Always use both together. They skip the automatic name matching, which is useful when the GPU name cannot be matched to exactly one product. See [Finding the product IDs](#finding-the-product-ids). |
| `-DriverPath <file>` and `-TargetVersion <version>` | Install a driver package you already downloaded instead of downloading one. `-TargetVersion` is the version of that package (format `596.58`) and is required, because the script compares it with the installed version before installing. The signature is still checked and the file is not deleted afterwards. |
| `-DownloadTimeoutMinutes <n>` | Total download time limit including retries. Default 90. |
| `-StallTimeoutSeconds <n>` | Retry when no data arrives for this long. Default 120. |

### Examples

Check what would happen (safe, nothing changes):

```powershell
.\scripts\Update-NVIDIADriver.ps1 -CheckOnly
```

Install the latest driver (elevated PowerShell):

```powershell
.\scripts\Update-NVIDIADriver.ps1
```

Clean install:

```powershell
.\scripts\Update-NVIDIADriver.ps1 -Clean
```

The GPU name cannot be matched, so supply the IDs yourself:

```powershell
.\scripts\Update-NVIDIADriver.ps1 -ProductSeriesId 127 -ProductId 1000
```

Use a package you downloaded earlier (for example on a slow or offline device):

```powershell
.\scripts\Update-NVIDIADriver.ps1 -DriverPath "C:\Temp\596.58-desktop-win10-win11-64bit-international-dch-whql.exe" -TargetVersion 596.58
```

Log to the Intune folder, or to a folder of your choice:

```powershell
.\scripts\Update-NVIDIADriver.ps1 -IntuneLog
.\scripts\Update-NVIDIADriver.ps1 -LogPath "D:\Logs"
```

The numbers in the examples are placeholders. Use your own values.

### Finding the product IDs

1. Run `.\scripts\Update-NVIDIADriver.ps1 -CheckOnly`.
2. Open the log (default `C:\ProgramData\NVIDIA-DriverAutoUpdate\Logs\NVIDIA-Driver.log`).
3. Look for the line `Lookup match: <name>; psid: <number>; pfid: <number>`. Those are the values.

If the script reports `Found N unique NVIDIA lookup matches`, the error lists the candidates as `name [psid/pfid]`. Pick the one that matches your GPU and pass its two numbers as `-ProductSeriesId` and `-ProductId`.

You can also look the IDs up manually on [nvidia.com/Download](https://www.nvidia.com/Download/index.aspx).

## Troubleshooting

Every run writes to the log (see [What the update script does](#what-the-update-script-does) for the location) and ends with `Finished with exit code N.` When the script stops, the cause is in the line before that. Common messages:

| Message in the log | Cause and fix |
| --- | --- |
| `No NVIDIA GPU detected.` | No device with PCI vendor ID `10DE` was found. In Intune, use the requirement script so these devices show as Not applicable. |
| `Multiple NVIDIA GPUs detected (...)` | More than one NVIDIA GPU is out of scope for automatic matching. Pass `-ProductSeriesId` and `-ProductId`. |
| `Found 0 unique NVIDIA lookup matches` or `Found N unique NVIDIA lookup matches` | The Windows GPU name maps to no product or to several products. Pass `-ProductSeriesId` and `-ProductId`, see [Finding the product IDs](#finding-the-product-ids). |
| `Administrator or SYSTEM rights are required to install.` | Run elevated, or as System in Intune. `-CheckOnly` works without admin rights. |
| `Run this script in 64-bit PowerShell.` | The script was started in 32-bit PowerShell. In Intune, set "Run script as 32-bit process" to No. |
| `Download did not complete within N minutes.` | Slow or interrupted link. The partial file is kept in `%ProgramData%\NVIDIA-DriverAutoUpdate\Cache` and resumed on the next run. Raise `-DownloadTimeoutMinutes` if needed. |
| `Invalid signature on ...` or `Unexpected signer on ...` | The downloaded package is not validly signed by NVIDIA Corporation. The script does not run it, and on the next run a cached copy that fails the check is deleted and downloaded again. If it persists, check for a proxy or filter that alters downloads. |
| `NVIDIA installation failed with exit code N.` | The NVIDIA installer itself failed. Extracted files are kept for inspection. Run with `-Clean`, or check whether another installation or a pending restart blocks it. |
| `Version not yet active. Expected after restart: ...` | Not an error. The driver is installed and becomes active after a restart (exit code `3010`). |

## When to use this vs. alternatives

Facts from each project's own page, read on 28 October 2026. Check the linked pages for the current state.

| Tool | Notable features | Good fit when |
| --- | --- | --- |
| This script | GPU detection by PCI vendor ID, Authenticode check, no downgrade, exit codes `0`/`1`/`3010`, download resume, Intune detection and requirement scripts | Managed devices, run as SYSTEM from Intune or another deployment tool |
| [lord-carlos/nvidia-update](https://github.com/lord-carlos/nvidia-update) | Checks for a newer driver and installs it, `-clean`, sample scheduled task command; needs 7-Zip or WinRAR | A single PC with a daily scheduled task |
| [ZenitH-AT/nvidia-update](https://github.com/ZenitH-AT/nvidia-update) | `-Silent`, `-Schedule` (weekly task), `-LogFilePath`, `-Clean`, optional components such as PhysX via a config file | A personal PC where you want to choose components |
| [Chocolatey `nvidia-display-driver`](https://community.chocolatey.org/packages/nvidia-display-driver) | Fixed driver version per package release, SHA-256 checksum, switches to skip components (`/NoAudio`, `/NoUSBC`) | Fleets already managed with Chocolatey |

## Intune example

See [intune/README.md](intune/README.md) for deploying this as a Win32 app with detection and requirement rules.

## Notes

- Windows 10/11, 64-bit only. Single NVIDIA GPU per device is in scope.
- Scripts are unsigned in this repository. Sign them with your own code-signing certificate if your execution policy requires it.
- Provided as is, test on a pilot group first.

## License

MIT. See [LICENSE](LICENSE). Original work copyright (c) 2022 Dmitry Nefedov.
