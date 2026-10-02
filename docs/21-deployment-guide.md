# Deployment guide

| | |
|---|---|
| **Document** | 21 · Deployment guide |
| **Version** | 0.1 |
| **Status** | Draft for review |
| **Date** | 29 September 2026 |
| **Owner** | Peter Pirisola |
| **Scope** | Product (client-neutral). Client specifics are kept in client notes. |
| **Classification** | Public copy (see the note below the table) |
| **Audience** | The client's IT team: endpoint, Microsoft 365 and Power BI administrators |
| **Related documents** | Technical requirements · System architecture · Data design · [Release test script](22-release-test-script.md) · [Quick guide](23-quick-guide.md) · ADR-21 · ADR-22 |

> **Public copy.** Published from D8A AI XYZ's internal document set on 2 October 2026. References to other numbered documents, architecture decision records (ADRs) and reviews point at the internal specification, which is not published; they are shown as plain text here.

## Purpose

This guide is for the IT team that installs Data Platform on a client's Windows PCs. It assumes you know Intune or Configuration Manager (SCCM) and Windows Installer, and nothing about Data Platform. It covers what you deploy, what must be in place first, the configuration file, the install, upgrade and removal, what users see, and where to look when something goes wrong.

Data Platform is a desktop Excel add-in. It adds a **Data Platform** tab to Excel, where users pick fields from approved Power BI semantic models and load them as Excel tables or PivotTables. Every query runs as the signed-in user, so their Power BI permissions, row-level security and object-level security apply. Nothing is hosted: the add-in talks only to Microsoft Entra ID, Power BI and, if you configure one, the SharePoint site that holds the central list.

`<version>` in this guide stands for the release number, for example `1.0.0`.

## 1. What you deploy

| File | For | Notes |
|---|---|---|
| `DataPlatform-<version>-x64.msi` | PCs with 64-bit Microsoft 365 Apps | Installs to `%ProgramFiles%\D8A\DataPlatform\` |
| `DataPlatform-<version>-x86.msi` | PCs with 32-bit Microsoft 365 Apps | Installs to `%ProgramFiles(x86)%\D8A\DataPlatform\` on 64-bit Windows |

- **One package per Office bitness.** Each installer refuses a PC whose Office is the other bitness (section 11). On a PC where no Office is found, **both** MSIs install: install only the package that matches the Office you will install there. With both installed, each user gets two add-in entries, one of the wrong bitness, and Excel complains about it at every start. Whether the tab appears once the matching Office is installed afterwards has not been tested, so install Office first.
- **Per machine.** Each MSI installs once per PC, for every user. There is nothing to install per user.
- **The same package for every client.** Nothing client-specific is inside the MSI. Your tenant, app registration and approved workspaces are in a configuration file you supply (section 3).
- **Unsigned in this release candidate.** The signed build follows once the code-signing certificate is in place (decision D-08). Until then, see section 10 if your Office policy requires signed add-ins.
- **With every release:** `DataPlatform-<version>.cdx.json`, the software bill of materials (CycloneDX), and `SHA256SUMS.txt`. Check the MSI you package against it, for example with `Get-FileHash DataPlatform-<version>-x64.msi` in PowerShell, before you upload it to Intune or SCCM.
- **Prerequisite checked, not installed.** The .NET 10 Desktop Runtime of the matching bitness must be on the PC first. The installer checks for it and stops with a message if it is missing; it does not install it.
- **Supported client:** Excel for Microsoft 365 (Microsoft 365 Apps for enterprise) on Windows. Excel for the web, Mac and mobile open the workbooks and show the values last saved in them.

The version is written in the file name and in the registry (section 4). Release builds are made from a tagged commit (`data-platform/v<version>`). A package whose version is `0.0.0` is an untagged development build: do not deploy it.

## 2. Prerequisites

| Prerequisite | What is needed | Who | Notes |
|---|---|---|---|
| .NET 10 Desktop Runtime | The Desktop Runtime (not only the .NET Runtime) of the same bitness as Office: x64 for 64-bit Office, x86 for 32-bit Office. Any .NET 10 update (10.0.x) works; a PC with only .NET 11 is refused. Download from <https://dotnet.microsoft.com/download/dotnet/10.0>. | Client IT | Deploy it as its own Intune or SCCM app and make it a dependency of Data Platform. On estates with both Office bitnesses, deploy both runtimes to the matching groups. |
| Power BI capacity and XMLA | The models are on Premium, Premium Per User or Fabric capacity, with the XMLA endpoint set to read (capacity setting), and the tenant setting that allows XMLA endpoints on. | Power BI administrator | Needed for every table. |
| Live connection setting (Matrix layout only) | Tenant setting "Users can work with semantic models in Excel using a live connection" (Export and sharing settings) on for Data Platform users. | Power BI administrator | Without it, the Table layout still works and the Matrix layout shows DP-E24. |
| App registration | One Microsoft Entra app registration with admin consent for the delegated Power BI scopes (and SharePoint read if the central list lives on SharePoint). | Entra administrator | Set up from the D8A onboarding pack; its ID goes into the configuration file. |
| Configuration file | `config.json` for your organisation: tenant, app registration and approved workspaces (section 3). | Client IT with D8A | Supplied at install time or by a separate script. |
| Model access | Users have Build permission on the models and the Viewer role in the workspace where row-level security must apply. | Model owners | Admin, Member and Contributor roles bypass row-level security. |

## 3. The configuration file

### 3.1 What it holds

`config.json` tells the add-in which tenant and app registration to sign in with and which workspaces are approved. The full schema is in Data design section 5.2; the main keys are:

| Key | Meaning |
|---|---|
| `schemaVersion`, `configVersion` | The schema version (1) and your own version label for the file, shown in the About window |
| `clientId`, `clientName` | A short identifier and the display name of your organisation |
| `ribbonLabel` | The name of the Excel tab (default "Data Platform") |
| `tenantId` | Your Microsoft Entra tenant ID |
| `appRegistrationId` | The application (client) ID of the app registration |
| `powerBiApi`, `xmlaBase` | The Power BI endpoints (the public cloud values unless D8A tells you otherwise) |
| `approvedWorkspaces` | The approved workspaces: each with its `id`, a `label`, and optionally a `description`, `environment` and `owner` shown on the workspace cards |
| `centralConfigUrl`, `allowedConfigHosts` | Optional: the address of a central list on SharePoint and the hosts it may come from |
| `nudgeRowThreshold`, `maxRowsBeforeWarning`, `logLevel` | Optional tuning: when the add-in suggests a filter, when it warns about size, and how much it logs |

D8A supplies a filled-in file with your values during onboarding and, on request, an example file.

### 3.2 Where it lives

`%ProgramData%\D8A\DataPlatform\config.json`, for every user on the PC. The installer sets a protected access list on `%ProgramData%\D8A` and `%ProgramData%\D8A\DataPlatform`: Administrators and SYSTEM have full control, Users can read, the owner is Administrators, and nothing is inherited from `%ProgramData%` (whose inherited entries would let users create files there). The folder's permissions stop standard users creating or changing files there. A file that existed before the first install is removed by the installer, with or without `CONFIGDIR` (section 3.3), so it cannot keep its own permissions; the route B script (section 3.4) also deletes any file already there before writing its own. The add-in never writes the file.

### 3.3 Route A: at install time with CONFIGDIR

Pass the folder that holds `config.json` as `CONFIGDIR`:

```
msiexec /i "DataPlatform-<version>-x64.msi" /qn /norestart CONFIGDIR="C:\Deploy\DataPlatform"
```

- `CONFIGDIR` is the **full path of a folder**, not of the file. The file in it must be called `config.json`.
- The install runs as SYSTEM, so the **computer account** must be able to read the folder. A UNC path such as `\\server\share\dataplatform` works only on domain-joined or hybrid-joined PCs where the computer account has read access to the share. On Entra-only PCs, use a local folder (section 4.2 shows how to ship the file inside the Intune package).
- If the folder has no `config.json`, the install stops with: "config.json was not found in the folder CONFIGDIR names (…). CONFIGDIR must be the full path of a folder holding config.json that this computer's account can read."

What the installer does with the file:

| Situation | What happens to `%ProgramData%\D8A\DataPlatform\config.json` |
|---|---|
| First install with `CONFIGDIR` | Copied from `CONFIGDIR`. A `config.json` already there (for example one created before the install) is replaced by yours. |
| Upgrade, a `config.json` is already there | Kept as it is, including any edits you made. `CONFIGDIR` is not copied; if it is passed, it must still hold a `config.json`, or the upgrade stops with the message above. |
| Upgrade, no `config.json` there, `CONFIGDIR` passed | Copied from `CONFIGDIR`. |
| Repair or reinstall | Never removed and never replaced, even when `CONFIGDIR` is passed again. |
| First install without `CONFIGDIR` | Nothing is copied. A `config.json` already there (for example one a user created before the install) is removed, as on a first install with `CONFIGDIR`. The add-in shows DP-E12 until a file is put there by route B. |
| Upgrade without `CONFIGDIR` | A `config.json` already there is kept; if there is none, nothing is copied. |
| Uninstall | Removed, with the `D8A` folders when they are empty. |

To change the configuration on PCs that already have it, use route B, or uninstall and install again with the new file.

### 3.4 Route B: a separate script

Any tool that runs as SYSTEM or an administrator can write the file instead: an Intune platform script, a Win32 app of its own, or an SCCM package. Run it **after** Data Platform is installed, so the folder already has its protected access list. The script first deletes any `config.json` already there, taking ownership of it if its own permissions refuse SYSTEM, so the new file is owned by Administrators or SYSTEM and inherits the folder's protected list. An example Intune script (run as SYSTEM, 64-bit PowerShell); paste your complete `config.json` between the `@'` lines (a file missing a required key, such as `clientName`, `powerBiApi` or `xmlaBase`, gives DP-E13):

```powershell
$folder = Join-Path $env:ProgramData 'D8A\DataPlatform'
$file = Join-Path $folder 'config.json'
$json = @'
<paste your complete config.json here>
'@
New-Item -ItemType Directory -Force -Path $folder | Out-Null
Remove-Item -Path $file -Force -ErrorAction SilentlyContinue
if (Test-Path $file) {
  # A file whose own permissions refuse SYSTEM: take ownership for Administrators, reset its permissions, then delete it.
  takeown /F $file /A | Out-Null
  icacls $file /reset | Out-Null
  Remove-Item -Path $file -Force
}
Set-Content -Path $file -Value $json -Encoding UTF8
```

The file written this way is removed on uninstall, like one copied by the installer.

### 3.5 When changes take effect

The add-in reads `config.json` when Excel starts, so a change takes effect at each user's next Excel start (FR-CFG-08). When `centralConfigUrl` is set, the approved-workspace list is read from that SharePoint file and refreshed in the background without a reinstall or an Excel restart; only `approvedWorkspaces`, the two row thresholds and `configVersion` can come from it (sign-in and endpoints always come from the local file).

## 4. Install with Intune

The console labels in sections 4 and 5 are as of the Intune and Configuration Manager consoles in September 2026; confirm them in your tenant. Group H of the release test script exercises these sections.

Create one **Windows app (Win32)** per bitness, from an `.intunewin` package holding the MSI (and, for route A with a local folder, `config.json` and a small install script).

### 4.1 Program

| Setting | Value |
|---|---|
| Install command | `msiexec /i "DataPlatform-<version>-x64.msi" /qn /norestart CONFIGDIR="<folder>"` (or `install.cmd`, section 4.2) |
| Uninstall command | `msiexec /x {<ProductCode>} /qn /norestart` (Intune fills in the product code from the MSI), or the UpgradeCode script in section 8.3 |
| Install behaviour | System |
| Device restart behaviour | Determine behaviour based on return codes |
| Return codes | The standard MSI codes: 0 success, 1707 success, 3010 soft reboot, 1641 hard reboot, 1618 retry. 3010 is expected when a user had Excel open with the add-in loaded (section 7). |
| Time | The install takes under a minute. |

Add a verbose log to the command when testing: `/l*v "%TEMP%\DataPlatform-install.log"`. The log's folder must already exist and must never be under `%ProgramData%\D8A`: that folder does not exist before the first install, so msiexec could not open the log and would stop (1622), and a log left there would keep the folder from being removed on uninstall. Under SYSTEM, `%TEMP%` is `C:\Windows\Temp`; where the command field does not expand variables, write that path in full.

### 4.2 Shipping config.json inside the package (Entra-only PCs)

Put `config.json` in a `config` folder next to the MSI before you build the `.intunewin` file, and use this `install.cmd` as the install command. `%~dp0` is the folder Intune extracted the package to:

```
msiexec /i "%~dp0DataPlatform-<version>-x64.msi" /qn /norestart CONFIGDIR="%~dp0config"
exit /b %ERRORLEVEL%
```

The file is copied to `%ProgramData%` during the install; the extracted package folder is removed by Intune afterwards.

### 4.3 Detection rule

Use a registry rule. The x86 package writes its values in the 32-bit registry view, so its rule must say so.

| Package | Key path | Value name | Detection method | Associated with a 32-bit app on 64-bit clients |
|---|---|---|---|---|
| x64 | `HKEY_LOCAL_MACHINE\SOFTWARE\D8A\DataPlatform` | `Version` | String comparison, Equals `<version>` | No |
| x86 | `HKEY_LOCAL_MACHINE\SOFTWARE\D8A\DataPlatform` (read by Windows from `SOFTWARE\WOW6432Node\D8A\DataPlatform`) | `Version` | String comparison, Equals `<version>` | **Yes** |

`Version` is the numeric version (for example `1.0.0`); detect on `Version`, never on `ConfigInstalled`. The same key also holds `BuildVersion` (the full build string, for example `1.0.0-rc.1`), `InstallFolder`, `Bitness`, `ConfigFolder` and sometimes `ConfigInstalled`. `ConfigInstalled` is present only after an install that copied `config.json` from `CONFIGDIR`: it is missing after an install without `CONFIGDIR` and disappears after an upgrade that kept an existing file, so its absence does not mean there is no configuration. Release candidates of one release share the numeric version, so a pilot that moves between candidates can detect on `BuildVersion` instead. The MSI product code rule that Intune offers also works, but it changes with every build.

### 4.4 Requirement rule: the Office bitness

Target each package at the PCs with the matching Office, using the same registry values the installer checks:

| Office | Key | Value | Requirement |
|---|---|---|---|
| Microsoft 365 Apps (Click-to-Run) | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration` | `Platform` | String equals `x64` (x64 package) or `x86` (x86 package); 64-bit view (Associated with a 32-bit app: No) |
| MSI-installed Office 2016 or later | `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\16.0\Outlook` | `Bitness` | String equals `x64` or `x86`, in both registry views: the 64-bit view (Associated with a 32-bit app: No) and, for 32-bit MSI-installed Office on 64-bit Windows, the 32-bit view (Associated with a 32-bit app: Yes; Windows reads it from `SOFTWARE\WOW6432Node\Microsoft\Office\16.0\Outlook`) |

The installer itself reads Click-to-Run's `Platform` first and, when it is absent, the MSI Office `Bitness` value in both registry views: the x64 package refuses when either view says `x86`; the x86 package installs when either view says `x86` and refuses when a view says `x64` and none says `x86`. Intune requirement rules must all be met, so one package cannot carry a rule for each view: where MSI-installed 32-bit Office is on 64-bit Windows, use the 32-bit view rule for the x86 package, or assign by device groups built by Office bitness, which works for every case.

### 4.5 Dependencies

Make the .NET 10 Desktop Runtime app of the same bitness a dependency, set to install automatically.

## 5. Install with Configuration Manager (SCCM)

Create an **application** with one deployment type per bitness, of type **Windows Installer (\*.msi file)**.

| Setting | Value |
|---|---|
| Installation program | `msiexec /i "DataPlatform-<version>-x64.msi" /qn /norestart CONFIGDIR="<folder>"` |
| Uninstall program | `msiexec /x {<ProductCode>} /qn /norestart` (filled in from the MSI) |
| Installation behaviour | Install for system; whether or not a user is logged on |
| Detection method | The MSI product code (filled in), or the registry setting of section 4.3. For the x86 package, tick "This registry key is associated with a 32-bit application on 64-bit systems". |
| Requirements | A global condition on the registry values of section 4.4 |
| Dependencies | The .NET 10 Desktop Runtime of the same bitness |
| Return codes | The defaults (3010 is a soft reboot) |

A UNC `CONFIGDIR` works here when the computer accounts can read the share; otherwise copy `config.json` with the content and point `CONFIGDIR` at the local content folder in a script, as in section 4.2.

## 6. What users see

- **The tab.** The installer registers a Windows Active Setup entry (`HKLM\SOFTWARE\Microsoft\Active Setup\Installed Components\{…}`, section 8.2). At each user's **next Windows sign-in**, Windows runs `D8A.DataPlatform.Register.exe --register` once for that user; it adds the add-in to that user's Excel add-in list (`HKCU\Software\Microsoft\Office\16.0\Excel\Options`, the next free `OPEN` value, other add-ins untouched). The Data Platform tab appears at their next Excel start.
- **Users signed in during the install** see the tab after they sign out and in again (or restart). To avoid that, run the registration for them in their own context (the script in section 8.1 with `--register` in place of `--unregister`; it waits for the helper and reports its exit code).
- **New users** of the PC get it at their first sign-in.
- **First use.** The first time a person uses the add-in, a short notice says some columns and rows may not be available to them because of row-level and object-level security. Before their first table load, a note says Excel may ask how to connect to Power BI: they choose **Organisational account** and sign in with their work account. Excel remembers it after that. A first matrix may show a Microsoft sign-in window once.
- **Excel's native query approval.** Excel asks users to approve each new Power BI query as a "native database query" unless the setting Data › Get Data › Query Options › Global › Security "Require user approval for new native database queries" is turned off. The first-use material tells users to untick it (decision D-15).
- **Nothing per user to install.** Sign-in is the user's own work account through Windows single sign-on.

## 7. Upgrade

- **Deploy the newer MSI the same way** (a new Intune app version with supersedence, or a new SCCM deployment type that supersedes the old one). The installer removes the old version and installs the new one in one step.
- **The configuration file is kept** (section 3.3).
- **Deploy when Excel is closed where you can**: in a maintenance window, or as a required Intune assignment scheduled out of working hours. If Excel was not running, the install returns 0 and the new version runs at the next Excel start.
- **Excel open during the install.** The installer never closes a user's Excel (decision D-20). If Excel had the add-in loaded, the files Excel holds locked (the add-in's `.xll` loader and the native sign-in library) are replaced at the next Windows restart: the install returns **3010** and Windows is not restarted. The add-in's managed assemblies are expected to be replaced at once (they are loaded from bytes); the loader and the native library stay locked until the restart; release test script step A22 records what actually loads. The Excel session that stays open keeps running the old version. **If the user closes and reopens Excel before restarting Windows, Excel may load a mix of old and new files**, and Data Platform may not start correctly: the user should restart Windows before using Data Platform again. Intune normally treats 3010 as a soft reboot and, with "Determine behaviour based on return codes", tells the user a restart is pending. D-20 will be reconsidered from the pilot's experience.
- **To close Excel instead**, add `MSIRESTARTMANAGERCONTROL=0` to the install command: Windows Installer's Restart Manager should then close Excel (the user may lose unsaved work) and normally restarts it after the install, so no mixed state arises (release test script step A23).
- **Each user's registration** runs once more at their next sign-in after an upgrade to a higher version; it finds the entry already there and changes nothing.
- **An uninstall followed by a reinstall of the same version** does not re-run the registration for users who already ran it: Windows remembers per user which Active Setup version has run, and that record outlives the uninstall. If their Excel entry was removed in between (section 8), those users have no tab. Run `--register` for each of them (the section 8.1 script with `--register` in place of `--unregister`), or deploy a higher version, which runs it again at their next sign-in.
- **Roll the new version out to everyone who shares workbooks at the same time.** From the release with stored-definition schema 1.3 (data design §6.3), the first refresh of a table through the new version makes its stored definition 1.3; a colleague still on the older version then gets DP-E23 ("made with a newer Data Platform") on Refresh Table and Edit for that table until they upgrade. Excel's own Refresh All keeps working for them.
- **Known first-open exception.** A workbook saved by 0.1.0-rc.1 still asks Excel to refresh each matrix when it opens. The first time it opens with this version, Excel refreshes the matrix once before the add-in's checks. The add-in then turns that setting off; the change is kept only when the workbook is saved, and if it cannot be made the add-in tries again at the next open. Where the add-in does not run the open (the person is not signed in, the configuration is not valid, or the workbook is no longer the active one), the setting stays on and Excel refreshes the matrix on that open too, until the next open where the add-in runs its checks (or the next refresh or edit of the matrix through the add-in). From the next saved open the add-in's checks run first (D-24).
- **Rebuilds of the same version** (release candidates) upgrade in place; `BuildVersion` tells them apart.
- **Downgrades are refused** with "A newer version of Data Platform is installed. Uninstall it first to go back." To roll back, uninstall, then install the older MSI. Users who had already been registered by the newer version keep their Excel entry if it is still there; if the tab is missing after a rollback, run `--register` for them (section 6).

## 8. Uninstall

The uninstall removes the program files, the `HKLM\SOFTWARE\D8A\DataPlatform` values, the Active Setup entry, `config.json` and the `D8A` folders when they are empty. It does not reach into each user's registry: the add-in entry in each user's Excel list stays, pointing at a file that has gone. At that user's next Excel start, Excel is expected to report the missing file once and remove the entry itself; release test script step A8 confirms this, and this guide is updated from its result. The recommended path is to remove the entry per user before the uninstall (8.1). If you later reinstall the same version, run `--register` for the users who had it, or deploy a higher version (section 7).

### 8.1 Removing the per-user entry first

Run this as an Intune remediation or platform script **in the user's context** ("Run this script using the logged-on credentials": Yes; 64-bit PowerShell) before the uninstall. It removes the entry for the signed-in user only: other profiles on the PC keep theirs, so it must run once for each user of the PC (for example a remediation assigned to users):

```powershell
foreach ($key in 'HKLM:\SOFTWARE\D8A\DataPlatform', 'HKLM:\SOFTWARE\WOW6432Node\D8A\DataPlatform') {
  $dp = Get-ItemProperty -Path $key -ErrorAction SilentlyContinue
  if (-not $dp) { continue }
  $loader = if ($dp.Bitness -eq 'x64') { 'D8A.DataPlatform64.xll' } else { 'D8A.DataPlatform.xll' }
  $exe = Join-Path $dp.InstallFolder 'D8A.DataPlatform.Register.exe'
  $xll = Join-Path $dp.InstallFolder $loader
  $p = Start-Process -FilePath $exe -ArgumentList '--unregister', ('"{0}"' -f $xll) -Wait -PassThru
  if ($p.ExitCode -ne 0) {
    Write-Output "Data Platform registration helper exit code $($p.ExitCode)"
    exit $p.ExitCode
  }
}
```

**Why `Start-Process -Wait`.** The helper is a Windows (GUI) program, not a console one, so that nothing flashes on screen when Active Setup runs it at sign-in. PowerShell and `cmd` do not wait for such a program unless told to, and its exit code is visible only when the caller waits: **0** done, **1** wrong arguments, **2** the change to the user's registry failed. Calling it with `&` would return at once, before the helper has changed anything, and report success whatever happened.

From a command prompt, the same step is:

```
start "" /wait "<install folder>\D8A.DataPlatform.Register.exe" --unregister "<install folder>\D8A.DataPlatform64.xll"
echo %ERRORLEVEL%
```

(`D8A.DataPlatform.xll` for the x86 package.) The helper removes only the Data Platform entry, keeps other add-ins' entries and renumbers the list without gaps. For `--register`, use the same script or command with `--register` in place of `--unregister`.

### 8.2 The fixed identifiers

These never change between versions:

| Package | UpgradeCode | Active Setup id |
|---|---|---|
| x64 | `{7A840BE2-B570-455F-B4F6-FA0542ACEEDD}` | `{3A8C9953-6EDA-4FCC-A634-88B235543225}` |
| x86 | `{1E9DD0AC-56C0-4C7B-91E1-5BE61D0E808D}` | `{40C547A1-99E7-4799-BB31-2673ECA0E6C0}` (under `WOW6432Node` on 64-bit Windows) |

### 8.3 Uninstalling by UpgradeCode

The product code changes with every build; the UpgradeCode does not. To remove whichever version is installed:

```powershell
$installer = New-Object -ComObject WindowsInstaller.Installer
foreach ($code in $installer.RelatedProducts('{7A840BE2-B570-455F-B4F6-FA0542ACEEDD}')) {
  Start-Process msiexec.exe -ArgumentList "/x $code /qn /norestart" -Wait
}
```

Use `{1E9DD0AC-56C0-4C7B-91E1-5BE61D0E808D}` for the x86 package.

## 9. Logs and diagnostics

| What | Where | Notes |
|---|---|---|
| The add-in's log | `%LOCALAPPDATA%\D8A\DataPlatform\logs\dataplatform-YYYYMMDD.log`, per user | One file a day, at most 14 files and 50 MB. Each line has the time (UTC), level, event ID and a correlation ID. No data values, tokens or query results. |
| The registration log | `%LOCALAPPDATA%\D8A\DataPlatform\logs\register.log`, per user | One line per run: `<UTC time> DP0453 <register or unregister> <outcome> index=<n>`. Outcomes: `added`, `present` (already there), `replaced-stale` (an old entry pointing at a missing file was replaced), `removed`, `absent` (nothing to remove), `ok` (for `--status`, with the number of entries as the index), or `failed-<error type>`. No path is written. The helper's exit code (0 done, 1 wrong arguments, 2 failure) is visible only when the caller waits for it (section 8.1). |
| The installer's log | Wherever `/l*v` points | Only when you ask for it. |
| Diagnostics zip | `%LOCALAPPDATA%\D8A\DataPlatform\diagnostics\diagnostics-<yyyyMMdd-HHmmss>.zip`, per user | Made by **Copy diagnostics** (below). The time in the name is UTC, to the second, so two clicks a few seconds apart make two files. |

**The Account menu.** The **Account** control on the Data Platform tab is a split button. Its top half signs the user in when nobody is signed in and the configuration is valid; otherwise it opens the About window (version, bitness, configuration source and version, and the signed-in account). Its arrow opens a menu:

| Menu item | Key tip in the open menu | What it does | Available |
|---|---|---|---|
| **Sign out** | O | Signs the user out of Data Platform on this PC | Only when signed in |
| **About Data Platform** | B | Opens the About window | Always |
| **Copy diagnostics** | C | Writes the diagnostics zip (below) | Always, also when not signed in and when the configuration has a problem |
| **Run diagnostics check** | K | Checks the configuration, the sign-in, REST access and XMLA access, and shows the result (below) | When the configuration is valid |

The top half's key tip is S (Alt, G, S). The whole control, menu included, is greyed out while a load or refresh runs, and after the add-in failed to start; then restart Excel, and if that does not help, collect the log files from the folders above by hand.

**Copy diagnostics.** From the Account menu, or from the **Copy diagnostics** button on the About window and on the windows shown when the configuration has a problem or the add-in is still starting. It writes a zip holding the last 7 days of the add-in's logs, `register.log` (whatever its age) and `environment.txt` (add-in version, bitness, Excel version, Windows version, configuration source and version, the install folder the add-in was loaded from, and the log folder's size in bytes; no account name), and puts the zip's path on the clipboard. From the menu, a short message says "Diagnostics saved to `<path>` · the path is on your clipboard"; if Windows refused the clipboard it says only "Diagnostics saved to `<path>`" (the log records DP0456), and the user copies the path from the message. The user pastes the path into the service desk ticket and attaches the file. If the zip cannot be written the add-in says so and the log records DP0454.

**Run diagnostics check.** From the Account menu. It runs four checks, one after another, behind a loading panel with **Cancel**, and never asks the user to sign in: it uses only a sign-in Windows already holds. The XMLA check runs a one-row query with a 30-second limit on the first model of the first approved workspace the user can reach. The result opens in a window with **Copy report** (the button reads "Copied" for 2 seconds) and **Close**; the user pastes it into the ticket. The report names the user's account, a workspace and a model, so it is shown, not logged: the log gets one DP0455 line with pass or fail per check and the time taken. The report starts "Data Platform diagnostics check, `<version>`, `<yyyy-MM-dd HH:mm>` UTC" and has one line per check, `PASS`, `FAIL` or `SKIP`, then the check's name and the detail:

| Line | When it passes | Other results and what they mean |
|---|---|---|
| Configuration | `<source>, version <v>, <n> approved workspaces`: where the approved list came from (as the About window names it), its `configVersion`, and how many workspaces it approves | `FAIL` with the configuration problem, as the tab's screentips show it (DP-E12, DP-E13): correct `config.json` (section 3). Normally not seen: Run diagnostics check is greyed out while the configuration is invalid, so the tab's message is what you see; this line appears only if the add-in's services are missing while the configuration counts as valid. |
| Sign-in | `Signed in as <account>` | `FAIL Sign-in: Not signed in: choose Sign in first`: nobody is signed in, or Windows needs the user to sign in again; the user chooses **Sign in**. `FAIL` with a message and `(DP-Exx, <status>)`: the sign-in was refused, for example by Conditional Access; the status is the sign-in (MSAL) error code. REST and XMLA are then `SKIP … Skipped: not signed in`. |
| REST access | `<a> of <n> approved workspaces accessible; <m> models in <workspace>` | `FAIL REST access: No approved workspace is accessible to this account`: the account has no role in any approved workspace. `FAIL` with `(DP-Exx, <HTTP status>)`: Power BI could not be reached or refused the call (proxy, firewall, section 11). XMLA is then `SKIP … Skipped: no approved workspace to test`. |
| XMLA access | `ok in <ms> ms (<workspace> / <model>)` | `FAIL` with `(DP-Exx, <XMLA error number>)`: the XMLA endpoint is off or not set to read, the workspace is not on Premium, PPU or Fabric capacity, or the account has no Build permission on the model (section 2). `SKIP … Skipped: <workspace> has no model to test`. |

On any line, `FAIL … Unexpected error (<type>)` is a failure the add-in did not expect. The run goes on, but the checks that depend on the failed one are skipped: after Sign-in, REST and XMLA read `Skipped: not signed in`; after REST, XMLA reads `Skipped: no approved workspace to test` (in this case those reasons only mean the earlier check failed). Only a Configuration failure leaves the other three checks running. Send the report and a Copy diagnostics zip to D8A. If the three live lines read `SKIP … Skipped: the configuration is not valid`, the add-in's services did not start with this configuration: restart Excel and check `config.json`. A check stopped with **Cancel** shows the lines finished so far and ends with "Cancelled". The Configuration line always finishes first, so a cancelled report holds at least that line; the message "Cancelled · nothing was changed" with no report is a safeguard that is not normally seen.

**Error messages.** Every error the add-in shows has a code `DP-Exx` and a **Show details** link. The details are: the error code; the **Status** (the HTTP status, the XMLA error number, the Excel (COM) error code or the sign-in (MSAL) error code, or "-"); the category; the correlation ID; the **Power BI request ID** (quote this to Microsoft support when a case needs them); the time in UTC; and the add-in version. **Copy details** puts them on the clipboard. They never include server message text or data. For a failure that may pass (the network, Power BI busy), the message offers **Try again**, which runs the same command again.

## 10. Trusted publisher (signed builds)

Once the MSIs and the add-in are signed (after D-08), PCs where the Office policy "Require that application add-ins are signed by Trusted Publisher" is on need the D8A publisher certificate in the **Trusted Publishers** store, deployed by Intune (a trusted certificate profile) or Group Policy. With that policy on, Office disables unsigned add-ins, so the unsigned release candidate cannot be used on those PCs: pilot it on devices where the policy is off, or wait for the signed build.

## 11. Troubleshooting

| Symptom | Likely cause | What to check |
|---|---|---|
| The install fails: "Data Platform needs the .NET 10 Desktop Runtime (64-bit)…" (or 32-bit) | The Desktop Runtime of that bitness is missing | Install the Desktop Runtime from <https://dotnet.microsoft.com/download/dotnet/10.0>, then install again. A PC with only .NET 11 is refused. |
| The install fails: "This installer is for 64-bit Office. Use the other Data Platform installer for the Office installed on this computer." (or 32-bit) | Wrong package for this PC's Office | Check `Platform` (Click-to-Run) or the Outlook `Bitness` value in both registry views (section 4.4) and deploy the other package. |
| The install fails: "config.json was not found in the folder CONFIGDIR names…" | `CONFIGDIR` is wrong, or SYSTEM cannot read it | Use a full folder path the computer account can read; the file must be called `config.json`. |
| The install fails: "A newer version of Data Platform is installed…" | A downgrade | Uninstall first, then install the older version (section 7). |
| Install returns 3010 | Excel had the add-in loaded | Expected. The new version runs after the next Windows restart (section 7). |
| The tab is missing | The runtime is missing; the wrong bitness; the user has not signed in since the install, so Active Setup has not run; the user's Excel entry was removed | Check the runtime and bitness; ask the user to sign out and in; read `register.log` (no line: Active Setup has not run for this user; `failed-…`: the helper could not write the user's registry); look for the add-in under File › Options › Add-ins › Excel Add-ins. Run `--register` for the user if needed (section 6); after an uninstall and a reinstall of the same version this is always needed (section 7). |
| Excel reports at every start that it cannot load a Data Platform add-in | Both MSIs were installed on a PC where no Office was found (section 1) | Uninstall the package whose bitness does not match Office, then run the section 8.1 script per user with `--register` for the loader that remains: it keeps the good entry and drops the stale one, whose file is gone (an `--unregister` after the uninstall would find only the package still installed and remove the wrong entry). |
| The tab shows but its buttons are disabled with DP-E12, "Data Platform isn't set up on this computer…" | `config.json` is not in `%ProgramData%\D8A\DataPlatform` | Supply it (section 3). Restart Excel. |
| DP-E13, "Data Platform's setup on this computer has a problem…" with a field name | `config.json` is not valid | Correct the field named in the message; the Account window names it too. Restart Excel. |
| DP-E10, "You're offline, or Power BI can't be reached…" | No network, or a proxy or firewall blocks Microsoft Entra ID or Power BI | Allow Microsoft Entra ID (`login.microsoftonline.com`), the Power BI service URLs Microsoft publishes (including `api.powerbi.com` and the XMLA endpoints) and, if used, the SharePoint host of the central list. Tables already in the workbook keep their data. |
| Excel asks how to connect to Power BI | Excel's own Power Query credential prompt, the first time a table loads on the PC | Choose **Organisational account** and sign in with the work account. It is asked once per user and PC. |
| Excel asks to approve a "native database query" at every load | Excel's native query approval setting | Users turn it off (section 6, decision D-15). |
| A user's tables will not load and the message does not make the cause clear | Configuration, sign-in, Power BI access or XMLA | Ask the user to run **Account › Run diagnostics check** and send the report (section 9); the first line that is not `PASS` names the problem. |
| After an uninstall, Excel says it cannot find the add-in file | The user's Excel entry outlived the uninstall | Excel is expected to remove the entry after showing the message once (being confirmed by release test script step A8). To avoid it, run `--unregister` per user before the uninstall (section 8.1). |

## Open questions

| ID | Question | Needed from | Needed by |
|---|---|---|---|
| OQ-01 | Is Active Setup acceptable on the client's estate, and do users signed in during the install accept seeing the tab only after their next sign-in (06 OQ-21, T-04)? | Client IT team | Before the pilot |
| OQ-02 | Which route will the client use for `config.json`: `CONFIGDIR` at install time, or a separate script? | Client IT team | Before the pilot |
| OQ-03 | Does the client's Office policy require signed add-ins on the pilot devices? If so, the pilot waits for the signed build (D-08). | Client IT team | Before the pilot |

## Change history

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 29 September 2026 | Peter Pirisola | First draft, for the release candidate of workstream 8 part 1 (ADR-21). |
| 0.2 | 29 September 2026 | Peter Pirisola | Review fixes: the verbose log in `%TEMP%`; Excel's removal of a stale entry hedged until step A8; a file planted before an install without `CONFIGDIR`, and route B deleting and re-owning an existing file; console labels to confirm; the mixed old and new files after a 3010 upgrade, and deploying with Excel closed; smaller wording fixes. |
| 0.3 | 29 September 2026 | Peter Pirisola | Whole-branch review fixes: the helper run with `Start-Process -Wait` (or `start /wait`) and its exit code checked; a stray `config.json` removed on every first install; `ConfigInstalled` not for detection; both MSIs install where no Office is found; the Office bitness read in both registry views; the assemblies' replacement after a 3010 hedged; a same-version reinstall needs `--register`; the diagnostics zip's name to the second, `register.log` and the log folder size. |
| 0.4 | 29 September 2026 | Peter Pirisola | Workstream 8 part 2 (ADR-22): the Account menu, Copy diagnostics from the menu (also signed out), Run diagnostics check and what its four lines mean; the SBOM and `SHA256SUMS.txt` with every release. |
| 0.5 | 29 September 2026 | Peter Pirisola | Review fixes: after an unexpected error in Sign-in or REST the checks that depend on it are skipped; the Configuration FAIL line and the no-report Cancel are normally not seen. |
| 0.6 | 30 September 2026 | Peter Pirisola | The diagnostics report's button is **Copy report** (plan 13, ADR-24). |
| 0.7 | 1 October 2026 | Peter Pirisola | §7: roll a release that writes stored-definition schema 1.3 out to everyone who shares workbooks at once (plan 14, task 2). |
| 0.8 | 1 October 2026 | Peter Pirisola | §7: the known first-open exception for a workbook saved by 0.1.0-rc.1 (plan 14, task 5, D-24). |
| 0.9 | 1 October 2026 | Peter Pirisola | §7: the first-open exception also holds for every open where the add-in does not run its checks, until the next one where it does (plan 14, the whole-branch review's fix round; Task 5 review, minor 5). |
