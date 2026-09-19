# winbox4-msi-creation

WiX source for repackaging the MikroTik WinBox 4 ZIP release as an MSI, suitable for
unattended deployment with Action1, Intune, SCCM or plain `msiexec`.

The package:

- installs `winbox.exe` into `%ProgramFiles%\WinBox` (per-machine, 64-bit),
- creates a **Desktop** shortcut in `C:\Users\Public\Desktop` (visible to every user on
  the machine, including profiles created after installation),
- creates a **Start Menu** shortcut in `C:\ProgramData\Microsoft\Windows\Start Menu\Programs`,
- supports major upgrades, so deploying a newer build replaces the previous one.

MikroTik does not ship WinBox as an MSI, and the binary is not redistributed here. You
download the official ZIP and build the package yourself.

---

## Prerequisites

### 1. .NET SDK

WiX v7 ships as a .NET global tool and needs the .NET SDK 6.0 or later.

```powershell
dotnet --version
```

If the command is not found:

```powershell
winget install Microsoft.DotNet.SDK.8
```

**Close and reopen your terminal afterwards.** The installer modifies `PATH` and the
change does not apply to already-open sessions.

### 2. WiX Toolset v7

```powershell
dotnet tool install --global wix
```

Again, **close and reopen the terminal** — the tool installer says so explicitly and
`wix` will not resolve until you do.

Verify:

```powershell
wix --version
```

> **Note on `candle` and `light`.** Those are WiX v3 tools and do not exist in v4 and
> later; v7 has a single `wix` command. The `.wxs` file in this repo uses the v4 schema
> namespace (`http://wixtoolset.org/schemas/v4/wxs`), which is still current in v7. It
> will **not** compile with WiX v3.

### 3. Accept the OSMF EULA

WiX v7 refuses to run until the Open Source Maintenance Fee EULA is accepted:

```
error WIX7015: You must accept the Open Source Maintenance Fee (OSMF) EULA to use WiX Toolset v7.
```

The EULA ID for v7 is `wix7`. Two options:

**Per command** (documented by FireGiant as the option intended for build scripts and CI):

```powershell
wix build -acceptEula wix7 ...
```

This flag applies only to that one invocation — it does not persist.

**Once per user, per machine:**

```powershell
wix eula accept wix7
```

This writes a marker file into the user profile so subsequent commands work without the
flag. This subcommand is reported in community issue threads rather than spelled out in
the FireGiant docs; if `wix eula` does not exist in your build, check `wix --help` and
stick with `-acceptEula`.

> **Licensing.** Accepting the EULA is not the same as paying. The source remains
> available under its LICENSE; the Open Source Maintenance Fee obliges organizations
> generating more than USD 10,000 in annual revenue (as defined by the OSMF terms) to
> sponsor the `wixtoolset` GitHub organization. The fee was introduced in WiX v6, but
> EULA enforcement at the CLI arrived in v7. If you need to avoid the gate entirely in
> automation, pin the last version before it:
>
> ```powershell
> dotnet tool install --global wix --version 6.0.1
> ```

---

## Building the MSI

1. Download the WinBox 4 ZIP from the official MikroTik download page.
2. Extract it and copy `winbox.exe` next to `winbox.wxs` in this repo's working
   directory. Confirm the file name matches what the `.wxs` expects:

   ```powershell
   Get-ChildItem *.exe
   ```

   If MikroTik ships it under a different name, update the `Source=` attribute on the
   `<File>` element.

3. Check the version. MSI compares versions numerically, so it must match the binary:

   ```powershell
   (Get-Item .\winbox.exe).VersionInfo.FileVersion
   ```

   Update `Version="4.4.0"` on the `<Package>` element if needed.

4. Build:

   ```powershell
   wix build winbox.wxs -arch x64 -o WinBox4.msi
   ```

   `-arch x64` is required — without it, `ProgramFiles64Folder` and
   `Bitness="always64"` fail validation.

   If you have not accepted the EULA persistently:

   ```powershell
   wix build -acceptEula wix7 winbox.wxs -arch x64 -o WinBox4.msi
   ```

---

## Versioning rules

MSI version handling has two behaviours worth knowing before you ship a second build:

- Use `major.minor.build` format. `4.4` alone is accepted, but write `4.4.0` explicitly
  so upgrade comparisons behave predictably.
- **The fourth field is ignored** when Windows Installer detects a major upgrade. If you
  rebuild the same WinBox version (a packaging fix, say), bump the third field
  (`4.4.1`) — bumping a fourth field will not trigger the upgrade.

Keep the `UpgradeCode` GUID stable across all versions of this package. Changing it
breaks the upgrade chain and leaves both versions installed side by side.

---

## Deployment with Action1

Action1 runs deployments as `SYSTEM`. This is why the package is per-machine: a
per-user MSI installing to the desktop silently lands in the SYSTEM profile and appears
to do nothing.

1. Upload `WinBox4.msi` to your Action1 software repository.
2. Create a **Deploy Software** action pointing at the MSI.
3. Installation parameters:

   ```
   /qn /norestart
   ```

Action1 invokes `msiexec` itself; you only supply the switches.

Manual equivalent, for testing on a single machine from an elevated prompt:

```powershell
msiexec /i WinBox4.msi /qn /norestart
```

To capture a verbose log while troubleshooting:

```powershell
msiexec /i WinBox4.msi /qn /norestart /l*v install.log
```

### Uninstalling

Find the ProductCode after a test install:

```powershell
Get-ChildItem HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall |
  ForEach-Object { Get-ItemProperty $_.PSPath } |
  Where-Object DisplayName -like 'WinBox*' |
  Select-Object DisplayName, DisplayVersion, PSChildName
```

`PSChildName` is the ProductCode. Then:

```powershell
msiexec /x "{PRODUCT-CODE-HERE}" /qn /norestart
```

Note that the ProductCode changes with every build. The `UpgradeCode` is the stable
identifier — use that one where your tooling supports it.

---

## Customising the package

**Additional files.** If the ZIP contains more than the executable, either add further
`<File Source="..." />` elements inside the same `<Component>`, or replace the component
contents with a wildcard harvest:

```xml
<Files Include="*" />
```

**Dropping a shortcut.** Remove the corresponding `<Shortcut>` element. Both shortcuts
are children of the `<File>` element, so removing one does not affect component
identity.

**Install location.** Change `Name="WinBox"` on the `<Directory>` element, or swap
`ProgramFiles64Folder` for `ProgramFiles6432Folder` if you ever need a 32-bit build
(drop `-arch x64` and `Bitness="always64"` in that case).

**Rebranding.** `Name` and `Manufacturer` on `<Package>` control what appears in
Programs and Features. `Language="1045"` is Polish; change it to `1033` for English.

---

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `candle : The term 'candle' is not recognized` | WiX v3 tooling; v7 uses `wix build` |
| `wix : The term 'wix' is not recognized` | Terminal opened before the tool install — reopen it |
| `error WIX7015` | OSMF EULA not accepted; see above |
| `error WIX0199 ... incorrect namespace` | The `.wxs` still uses the v3 namespace |
| Installs, but nothing appears on the desktop | Per-user package deployed as SYSTEM; this repo's package is per-machine and does not have that problem |
| ICE validation failures | Add `-sval` to `wix build` to skip validation entirely (v7 has no per-ICE suppression equivalent to v3's `-sice:`) |

---

## Licence

The `.wxs` file in this repo is provided as-is. WinBox is MikroTik's software and is
subject to MikroTik's own licence terms; download it from the official source. WiX
Toolset is subject to its LICENSE and the Open Source Maintenance Fee terms described
above.
