# Data Platform for Excel

An Excel add-in by D8A AI XYZ. It adds a **Data Platform** tab to Excel for Microsoft 365. A user picks an approved Power BI workspace, a semantic model, a perspective, fields and filters, and gets a normal Excel table or a PivotTable (the Matrix layout) that refreshes at any time, always as the person using Excel. Row-level and object-level security in Power BI apply to every load and refresh. Nothing is hosted: there is no server, service or database between Excel and Power BI.

This repository publishes the **installer releases** and the **user and IT documentation**. The source code is developed in a private D8A repository.

## Download

Releases are on the [Releases page](../../releases). Each release carries:

| File | For |
|---|---|
| `DataPlatform-<version>-x64.msi` | PCs with 64-bit Microsoft 365 Apps (most PCs) |
| `DataPlatform-<version>-x86.msi` | PCs with 32-bit Microsoft 365 Apps |
| `DataPlatform-<version>.cdx.json` | The software bill of materials (CycloneDX) |
| `SHA256SUMS.txt` | SHA-256 checksums of the files above |

Excel's bitness is under File › Account › About Excel. Each MSI refuses a PC whose Office is the other bitness.

The current release is a **release candidate**, unsigned. Windows SmartScreen may say "Windows protected your PC"; More info › Run anyway continues. To check a download, compare its hash with `SHA256SUMS.txt`:

```powershell
Get-FileHash .\DataPlatform-<version>-x64.msi -Algorithm SHA256
```

## What a PC needs

- Windows 10 or 11 with Excel for Microsoft 365 (the desktop app). Excel for the web, Mac and mobile open the workbooks and show the values last saved in them.
- The [.NET 10 Desktop Runtime](https://dotnet.microsoft.com/download/dotnet/10.0) of the same bitness as Office (x64 for 64-bit Office, x86 for 32-bit Office). The installer checks for it and stops with a message if it is missing; it does not install it.
- Local administrator rights for the install. It is a per-machine install; every user of the PC gets the tab.
- A **configuration file** `config.json` for your organisation: the tenant, the app registration and the approved workspaces. Without it the add-in has nothing to connect to. Your IT team prepares it with D8A; the [deployment guide](docs/21-deployment-guide.md) describes it and the two ways to supply it.

## Install

From an administrator PowerShell, in the folder holding the MSI, with Excel closed:

```powershell
msiexec /i "DataPlatform-<version>-x64.msi" /qn /norestart CONFIGDIR="C:\Deploy\DataPlatform" /l*v "$env:TEMP\DataPlatform-install.log"
```

`CONFIGDIR` is the full path of a folder that holds `config.json`. Double-clicking the MSI also works, but a double-click cannot pass `CONFIGDIR`, so afterwards copy `config.json` into `%ProgramData%\D8A\DataPlatform\` yourself as an administrator.

Every user gets the tab after their next Windows sign-in. To have it at once, run the registration once as yourself:

```powershell
& "C:\Program Files\D8A\DataPlatform\D8A.DataPlatform.Register.exe" --register "C:\Program Files\D8A\DataPlatform\D8A.DataPlatform64.xll"
```

Start Excel: the Data Platform tab is there (key tip: Alt, G). Sign in with Account; the Windows account picker appears. Turn Excel's native-query approval off once (Data › Get Data › Query Options › Security, untick "Require user approval for new native database queries"), or every load asks.

A newer build installs over the old one with the same command and keeps `config.json`. Uninstall with `msiexec /x "DataPlatform-<version>-x64.msi" /qn /norestart` or from Settings › Apps. Intune and SCCM deployment, upgrades, the registry keys and troubleshooting are in the deployment guide.

## Documentation

| Document | Who it is for |
|---|---|
| [Quick guide](docs/23-quick-guide.md) | Users and trainers: the tab, New Table in six steps, refreshing, the filter cells, what "as you" means, what to do when something goes wrong |
| [Deployment guide](docs/21-deployment-guide.md) | IT: prerequisites, the configuration file, install routes (command line, Intune, SCCM), upgrade, uninstall, what the installer writes, troubleshooting |
| [Release test script](docs/22-release-test-script.md) | Testers and IT: the checks per release, grouped from install to data safety |
| [Accessibility test script](docs/24-accessibility-test-script.md) | Testers: keyboard, Narrator, contrast and zoom checks |
| [Installer identifiers](docs/installer-identifiers.md) | IT and packagers: the fixed UpgradeCodes and Active Setup ids, `CONFIGDIR` behaviour |

These are public copies of D8A's internal documents. References to other numbered documents and decision records point at the internal specification, which is not published.

## Status

Release candidate for pilot use. Known limits of this build:

- Unsigned installer (code signing is not yet decided), so SmartScreen warns on download and double-click.
- The MSI is 0.1.0 for every release candidate, so a later candidate installs over an earlier one.
- Logs are local only, at `%LOCALAPPDATA%\D8A\DataPlatform\logs`; they never hold data values, names or tokens. Account › Copy diagnostics puts the recent log and the version on the clipboard for a support request.

Problems and questions: open an issue in this repository. Please include the DP-E code the add-in showed and the diagnostics text from Account › Copy diagnostics.

## Licence

No licence has been chosen yet. Until one is, the software and documents here are © 2026 D8A AI XYZ, all rights reserved; they may be downloaded and used for evaluation. "D8A AI XYZ" and the D8A mark are trademarks of D8A AI XYZ.
