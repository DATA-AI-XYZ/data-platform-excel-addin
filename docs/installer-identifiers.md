# Installer identifiers and behaviour

> **Public copy.** Published from D8A AI XYZ's internal installer notes on 2 October 2026, for IT teams and packagers.

Data Platform ships as one per-machine MSI per Office bitness, built with WiX Toolset v5. The installers are unsigned until code signing is decided.

## Fixed identifiers

These never change. A new UpgradeCode would stop major upgrades from finding the installed product; a new Active Setup id would re-run registration for every user and leave the old entry behind.

| Bitness | UpgradeCode | Active Setup id |
| --- | --- | --- |
| x64 (64-bit Office) | `7A840BE2-B570-455F-B4F6-FA0542ACEEDD` | `{3A8C9953-6EDA-4FCC-A634-88B235543225}` |
| x86 (32-bit Office) | `1E9DD0AC-56C0-4C7B-91E1-5BE61D0E808D` | `{40C547A1-99E7-4799-BB31-2673ECA0E6C0}` |

## Versions

The MSI's ProductVersion is the numeric part of the release version (`0.1.0-rc.2` installs as `0.1.0`); the full string is kept as `BuildVersion` in the MSI's properties and shown by Account › About Data Platform. Pre-release builds share the release's numeric version, so the MSI allows same-version upgrades: a later release candidate replaces the installed one instead of installing beside it.

## What the installer puts on the PC

| Item | Where |
|---|---|
| The add-in (`D8A.DataPlatform64.xll` or `D8A.DataPlatform.xll`) and its files | `%ProgramFiles%\D8A\DataPlatform\` (x86 package on 64-bit Windows: `%ProgramFiles(x86)%\D8A\DataPlatform\`) |
| The registration helper `D8A.DataPlatform.Register.exe` | The same folder; Active Setup runs it once per user at their next sign-in, so each user gets the tab |
| The organisation's configuration | `%ProgramData%\D8A\DataPlatform\config.json`, readable by every user, writable by administrators and SYSTEM only |
| Per-user logs and settings | `%LOCALAPPDATA%\D8A\DataPlatform\` |

## CONFIGDIR

The package holds no organisation configuration. IT passes the full path of the folder holding `config.json`; the computer account must be able to read it (an Intune install runs as SYSTEM, so an Entra-only device needs a local path):

```
msiexec /i DataPlatform-<version>-x64.msi CONFIGDIR="C:\Deploy\DataPlatform" /qn /norestart
```

- If `CONFIGDIR` is given and the folder holds no `config.json`, the install stops with a message naming the folder.
- **First install:** a `config.json` already in `%ProgramData%\D8A\DataPlatform` (for example one a user created before the install) is removed on every first install, with or without `CONFIGDIR`. With `CONFIGDIR`, its `config.json` is then copied there; without it the folder is left empty for IT to fill afterwards.
- **Upgrade:** an existing `config.json` is kept, so a file IT may have edited is never replaced. If there is none and `CONFIGDIR` is passed, it is copied.
- **Files in use:** with Excel open the install returns 3010 and the new files take effect after a Windows restart; the installer never closes Excel and never restarts the PC on its own.

## Prerequisite check

The installer checks for the .NET 10 Desktop Runtime of its own bitness and stops with a message when it is missing; it does not install it. A PC with only a later major version (for example .NET 11) is refused, because the add-in targets .NET 10.

## Bitness check

Each package refuses a PC whose Office is the other bitness. On a PC where no Office is found, both packages install; install only the one that matches the Office that will be installed.
