# Term Sheet server — operator guide

> Verified against the current bootstrap, updater, release layout, and service installer on
> 26 August 2026.

This is for the person installing and running the Term Sheet server on your own Windows machine.
It covers installation, day-to-day operation, and what to do when something goes wrong. It does not
assume access to the vendor's source code or internal engineering documentation — you shouldn't need
either.

If anything here doesn't match what you're seeing, stop and contact the vendor rather than guessing.

## What setup handles automatically

The supported path is one verified bootstrap command on the server and one installer on each
employee computer. After the prerequisite Windows packages and organization-specific choices are
available, no one manually edits a hosts file, creates an application database, writes
`server.env`, generates application keys, registers a service, or starts a background process.

| Area | Automatic behavior |
| --- | --- |
| Server DNS and HTTPS | Setup configures Caddy, opens only HTTPS, keeps a localhost console URL, detects the server LAN address, and writes `server-host.txt` for LAN-only employee installs. |
| Server secrets and keys | Setup generates `AUTH_SECRET`, creates the document master key in DPAPI or Key Vault, protects configuration ACLs, and creates the requested custodian escrow packages. Provider API keys are entered later in Settings and stored encrypted. |
| Database | Setup configures the existing PostgreSQL 18 service, roles, database, grants, migrations, loopback-only listener, and generated application credential. |
| Services and updates | Setup installs and starts the web, extraction, backup, HTTPS proxy, updater, and rollback processor with automatic restart/startup. |
| Employee server mode | The installer copies the server URL and, for LAN DNS, validates and installs the generated hostname mapping automatically. A managed Tailscale build installs its pinned client and organization policy automatically. |
| Employee local mode | The launcher contains PostgreSQL 18, generates its private credential and encryption key, initializes/migrates the local database, and starts/stops it with the launcher. |

Human input is intentionally limited to security or infrastructure decisions that must not be
guessed: the Windows service credential, PostgreSQL administrator password, public hostname,
escrow certificates (or explicit deferred-escrow approval), BitLocker recovery-key custody, and
approved provider API keys. Node.js 24, PostgreSQL 18, NSSM, and Caddy are machine prerequisites;
Step 1 provides verified installation commands because enterprise package policy and the PostgreSQL
administrator password cannot safely be invented by the application.

Once installed, Windows starts the server automatically after reboot. To start or repair it on
demand, open elevated PowerShell in the retained bootstrap directory and run:

```powershell
.\repair-windows-server.ps1 -AcceptSafeRepairs
Start-Process "C:\TermSheet\Client Files\Term Sheet Extractor - Server.url"
```

## What you're installing

A self-contained Windows server: a web service, a background worker, a reverse proxy (Caddy) that
terminates HTTPS, and a small updater that checks for new versions on a timer and applies them
automatically. Everything the server ever runs is cryptographically signed by the vendor; the server
verifies that signature before installing anything, every time, including its own future updates.

Your database, documents, and configuration live outside the installed application entirely, so an
update — or a rollback from a bad one — never touches them.

Setup also creates the mandatory launcher handoff. A browser shortcut is added on the server
computer, while `Client Files` receives one checksummed Tauri employee installer containing both
**Sign in to the office server** and **Work on this computer**. The trusted release workstation
builds that executable before publication; server setup verifies and packages it without receiving
source code or build credentials. Distribution to employee computers is handled separately.

## Step 1: Before you start

Use a supported x64 Windows server, assign it a fixed LAN address, and point the intended DNS name
at that address. Normally, two people should also be available to hold escrow certificates. If the
organization has not appointed them yet, Step 7 provides an explicit temporary deferred-escrow
procedure.

Open **Windows PowerShell as Administrator**. All commands in this step run in that elevated window.
Confirm it is elevated before installing anything:

```powershell
$principal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
if (-not $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
  Write-Host "This window is not elevated. Accept the Windows prompt to open an administrator window."
  Start-Process powershell.exe -Verb RunAs
  return
}
```

The bootstrap maintains `%ProgramData%\WinnerZone\TermSheet\installation\installation-state.json`
as an atomic installation journal. A failed run records its exact phase, stable `TSE-*` diagnostic
code, transcript, and safe-resume status; rerun the same Step 7 command after correcting the named
condition. To run the shared installed diagnostic and produce a redacted ZIP without documents,
configuration secrets, tokens, or database contents:

```powershell
.\repair-windows-server.ps1 -AcceptSafeRepairs -CreateSupportBundle
# or bundle only:
.\export-installation-support-bundle.ps1
```

If elevation is declined, nothing is changed. Accept it later and restart Step 1; every installation
and check below is safe to rerun.

### Step 1.1: Install Node.js 24 x64

Node.js is needed only to verify and install the first release. Installed releases use their own
bundled, tested `runtime\node.exe`. Do not install npm packages, Git, or the source repository on the
server.

On Windows Server 2025 with WinGet available, install the version-specific Node 24 package. Do not
use `OpenJS.NodeJS.LTS`, because that package moves to a new major when the active LTS line changes.

```powershell
winget source update
winget install --id OpenJS.NodeJS.24 --exact --source winget --architecture x64 --scope machine `
  --accept-source-agreements --accept-package-agreements

# Refresh PATH in this PowerShell window, then verify both version and architecture.
$env:Path = [Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + `
  [Environment]::GetEnvironmentVariable("Path", "User")
node.exe --version
node.exe -p "process.arch"
```

The version must be `v24.` or newer and the architecture must be `x64`. If `winget.exe` is not
available (for example, on Windows Server 2022), download the **Windows x64 MSI for Node.js 24** from
the [official Node.js download page](https://nodejs.org/en/download), run the MSI as administrator,
accept the default machine-wide PATH option, open a new elevated PowerShell window, and run the two
verification commands above.

If WinGet says Node.js 24 is already installed, do not uninstall it: refresh `PATH` and run the two
checks again. If either check is still wrong, repair the existing Node.js 24 x64 installation from
Windows Installed Apps, then rerun Step 1.1.

### Step 1.2: Install and configure PostgreSQL 18 x64

PostgreSQL 18 is the supported database major and must be installed with its command-line tools.
On Windows Server 2025 with WinGet available, launch the official EDB installer with:

```powershell
winget install --id PostgreSQL.PostgreSQL.18 --exact --source winget --architecture x64 `
  --interactive --accept-source-agreements --accept-package-agreements
```

In the installer:

1. Keep the installation directory as `C:\Program Files\PostgreSQL\18`.
2. Install **PostgreSQL Server** and **Command Line Tools**. pgAdmin is optional; Stack Builder is
   not required by this application.
3. Keep the database data directory on the protected `C:` drive unless your organization's storage
   policy specifies another protected volume.
4. Set and securely record a password for the `postgres` database administrator.
5. Keep port `5432` and complete the installation. Do not expose this port to the LAN.

If WinGet is unavailable, download and run the PostgreSQL 18 x64 EDB installer linked from the
[official PostgreSQL Windows installers page](https://www.postgresql.org/download/windows/), using
the same selections above.

If PostgreSQL 18 is already present, keep it and continue; the Step 7 bootstrap helper discovers and
repairs the existing cluster. If the installer reports an incomplete installation, use its Repair option,
confirm `C:\Program Files\PostgreSQL\18\bin\psql.exe` exists, then rerun this step.

The default server configuration relies on BitLocker for document files stored under
`C:\TermSheetData`. Before continuing, enable BitLocker on `C:`, escrow its recovery key outside this
server through the organization's approved process, wait for encryption to reach 100%, and verify:

```powershell
$volume = Get-BitLockerVolume -MountPoint C:
$volume | Format-List MountPoint,VolumeStatus,ProtectionStatus,EncryptionPercentage
if ($volume.VolumeStatus -ne "FullyEncrypted" -or $volume.ProtectionStatus -ne "On" -or
    $volume.EncryptionPercentage -ne 100) {
  throw "C: must be fully encrypted and protected before Step 7."
}
```

Database configuration happens with the verified `configure-postgresql18.ps1` helper during
Step 7. Do not manually create roles or a database here. The helper handles fresh installations,
safe reruns, existing login or non-login roles, forgotten application-role passwords, a database
already owned by `term_sheet_extractor_admin`, the older `term_sheet_app` ownership layout, and a
database owned by another administrator. It also corrects runtime grants, the firewall rule, service
startup, loopback binding, and ownership of pre-existing application objects in the dedicated
`public` schema. Password mistakes prompt again instead of terminating setup.

### Step 1.3: Install the remaining prerequisites

Install [NSSM](https://nssm.cc/) and [Caddy](https://caddyserver.com/) from packages approved by your
organization. Put their x64 executables at the following paths, or substitute the approved absolute
paths in every later command. On a fresh server, the following elevated PowerShell installs the
modern-Windows-compatible NSSM 2.24-101 x64 build and the latest stable Caddy x64 build. It verifies
the NSSM digest published on its download page and Caddy's release-specific SHA-512 checksum before
copying either executable. The block always downloads fresh archives. On an existing installation it
stops every service whose executable path uses this NSSM copy, replaces both tools, and restarts only
the services that were running beforehand. Expect a brief application outage while the files are
replaced. If other applications share `C:\tools\nssm.exe`, schedule the command in an approved
maintenance window because those services are restarted too.

```powershell
$toolsRoot = "C:\tools"
$downloadRoot = Join-Path ([IO.Path]::GetTempPath()) `
  ("term-sheet-prerequisites-" + [guid]::NewGuid().ToString("N"))
New-Item -ItemType Directory -Path $toolsRoot, $downloadRoot -Force | Out-Null
$nssmDestination = Join-Path $toolsRoot "nssm.exe"
$caddyDestination = Join-Path $toolsRoot "caddy.exe"

# NSSM 2.24-101 x64. Do not use the older 2.24 build on modern Windows Server.
$nssmVersion = "2.24-101-g897c7ad"
$nssmArchive = Join-Path $downloadRoot "nssm-$nssmVersion.zip"
Invoke-WebRequest -Uri "https://nssm.cc/ci/nssm-$nssmVersion.zip" `
  -OutFile $nssmArchive -UseBasicParsing -ErrorAction Stop
$expectedNssmSha1 = "ca2f6782a05af85facf9b620e047b01271edd11d"
$actualNssmSha1 = (Get-FileHash -LiteralPath $nssmArchive -Algorithm SHA1).Hash.ToLowerInvariant()
if ($actualNssmSha1 -ne $expectedNssmSha1) {
  throw "NSSM archive checksum mismatch; do not install it."
}
$nssmExtract = Join-Path $downloadRoot "nssm"
Expand-Archive -LiteralPath $nssmArchive -DestinationPath $nssmExtract -ErrorAction Stop
$newNssm = Join-Path $nssmExtract "nssm-$nssmVersion\win64\nssm.exe"

# Latest stable Caddy x64, verified against its GitHub release checksums file.
$caddyRelease = Invoke-RestMethod `
  -Uri "https://api.github.com/repos/caddyserver/caddy/releases/latest" -ErrorAction Stop
$caddyVersion = $caddyRelease.tag_name.TrimStart("v")
$caddyAssetName = "caddy_${caddyVersion}_windows_amd64.zip"
$caddyAsset = $caddyRelease.assets |
  Where-Object name -eq $caddyAssetName | Select-Object -First 1
$caddyChecksums = $caddyRelease.assets |
  Where-Object name -eq "caddy_${caddyVersion}_checksums.txt" | Select-Object -First 1
if (-not $caddyAsset -or -not $caddyChecksums) {
  throw "The latest Caddy release does not contain the expected Windows x64 assets."
}
$caddyArchive = Join-Path $downloadRoot $caddyAssetName
$caddyChecksumsPath = Join-Path $downloadRoot $caddyChecksums.name
Invoke-WebRequest -Uri $caddyAsset.browser_download_url `
  -OutFile $caddyArchive -UseBasicParsing -ErrorAction Stop
Invoke-WebRequest -Uri $caddyChecksums.browser_download_url `
  -OutFile $caddyChecksumsPath -UseBasicParsing -ErrorAction Stop
$checksumLine = Get-Content -LiteralPath $caddyChecksumsPath |
  Where-Object { $_ -match "^[0-9a-fA-F]{128}\s+\*?$([regex]::Escape($caddyAssetName))$" } |
  Select-Object -First 1
if (-not $checksumLine) { throw "A valid SHA-512 entry for the Caddy archive was not found." }
$expectedCaddySha512 = ($checksumLine -split "\s+")[0].ToLowerInvariant()
$actualCaddySha512 = (Get-FileHash -LiteralPath $caddyArchive -Algorithm SHA512).Hash.ToLowerInvariant()
if ($actualCaddySha512 -ne $expectedCaddySha512) {
  throw "Caddy archive checksum mismatch; do not install it."
}
$caddyExtract = Join-Path $downloadRoot "caddy"
Expand-Archive -LiteralPath $caddyArchive -DestinationPath $caddyExtract -ErrorAction Stop
$newCaddy = Join-Path $caddyExtract "caddy.exe"

# Windows locks an NSSM executable while any service hosted by that copy is running.
# Download and verify everything first, then keep the service outage as short as possible.
$escapedNssmPath = [regex]::Escape([IO.Path]::GetFullPath($nssmDestination))
$affectedServices = @(
  Get-CimInstance Win32_Service -ErrorAction Stop |
    Where-Object { $_.PathName -match "(?i)$escapedNssmPath(?:`"|\s|$)" }
)
$servicesToRestart = @($affectedServices | Where-Object State -eq "Running")

try {
  foreach ($service in $servicesToRestart) {
    Write-Host "Stopping $($service.Name) so the prerequisite executables can be replaced..."
    Stop-Service -Name $service.Name -Force -ErrorAction Stop
  }
  foreach ($service in $servicesToRestart) {
    (Get-Service -Name $service.Name -ErrorAction Stop).WaitForStatus(
      [System.ServiceProcess.ServiceControllerStatus]::Stopped,
      [TimeSpan]::FromSeconds(60)
    )
  }

  Copy-Item -LiteralPath $newNssm -Destination $nssmDestination -Force -ErrorAction Stop
  Copy-Item -LiteralPath $newCaddy -Destination $caddyDestination -Force -ErrorAction Stop
  Unblock-File $nssmDestination, $caddyDestination -ErrorAction Stop
}
finally {
  foreach ($service in $servicesToRestart) {
    if ((Get-Service -Name $service.Name -ErrorAction SilentlyContinue).Status -ne "Running") {
      Write-Host "Restarting $($service.Name)..."
      Start-Service -Name $service.Name -ErrorAction Continue
    }
  }
}

Remove-Item -LiteralPath $downloadRoot -Recurse -Force -ErrorAction SilentlyContinue
```

NSSM 2.24 prints a plausible version inside its error/usage banner but exits with code 1 and fails
Steps 4 and 7. The supported build must return exit code 0 for `version`. Verify both installations:

```powershell
Get-Item C:\tools\nssm.exe, C:\tools\caddy.exe
& C:\tools\nssm.exe version
if ($LASTEXITCODE -ne 0) { throw "NSSM version check failed." }
& C:\tools\caddy.exe version
if ($LASTEXITCODE -ne 0) { throw "Caddy version check failed." }
```

By default, use the currently signed-in Windows account and retain its credential in this PowerShell
session. Reading the identity from Windows works for both local and domain sign-ins and avoids asking
the client to create another account solely for setup:

```powershell
$serviceUser = [Security.Principal.WindowsIdentity]::GetCurrent().Name
do {
  try {
    $serviceAccount = New-Object Security.Principal.NTAccount($serviceUser)
    $serviceSid = $serviceAccount.Translate([Security.Principal.SecurityIdentifier]).Value
    $serviceCredential = Get-Credential -UserName $serviceUser `
      -Message "Enter the current Windows account password used to run the Term Sheet services"
    if (-not $serviceCredential) { Write-Warning "No credential was entered." }
  } catch {
    Write-Warning "'$serviceUser' did not resolve: $($_.Exception.Message)"
    $serviceUser = Read-Host "Enter COMPUTERNAME\username or DOMAIN\username, or press Enter to stop"
    if (-not $serviceUser) { return }
  }
} until ($serviceSid -and $serviceCredential)
$serviceSid
```

If the signed-in account is a domain account, set `$serviceUser` to its real `DOMAIN\username`
instead. A dedicated service account or gMSA remains supported when company policy requires one;
set `$serviceUser` and `$serviceCredential` to that approved identity. The bootstrap resolves the
account to a SID before changing files or services, so a wrong account name fails before installation.
If the password is rejected during Step 7, run this credential block again and rerun the same
bootstrap command; no partial service installation needs to be removed first.

## Step 2: Download and extract the bootstrap package

The distribution repository root intentionally contains only the update channel, signing key, and
this guide. The PowerShell scripts are in the versioned bootstrap ZIP attached to the current server
release. If `preflight-connectivity.ps1` is already beside this guide, you are reading the extracted
copy and can continue to Step 3. Otherwise, open PowerShell in the directory where you want to keep
the installer and run:

```powershell
$bootstrapUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.3.9/term-sheet-bootstrap-0.3.9.zip"
$bootstrapChecksumUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.3.9/term-sheet-bootstrap-0.3.9.zip.sha256"
$downloadRoot = Join-Path $PWD "term-sheet-bootstrap-0.3.9-download"
$bootstrapZip = Join-Path $downloadRoot "term-sheet-bootstrap-0.3.9.zip"
$bootstrapChecksum = "$bootstrapZip.sha256"

$packageDirectory = Join-Path $downloadRoot "term-sheet-bootstrap-0.3.9"
New-Item -ItemType Directory -Path $downloadRoot -Force | Out-Null
for ($attempt = 1; $attempt -le 3; $attempt++) {
  try {
    Invoke-WebRequest -Uri $bootstrapUrl -OutFile "$bootstrapZip.part" -UseBasicParsing -ErrorAction Stop
    Move-Item -LiteralPath "$bootstrapZip.part" -Destination $bootstrapZip -Force
    Invoke-WebRequest -Uri $bootstrapChecksumUrl -OutFile $bootstrapChecksum -UseBasicParsing -ErrorAction Stop
    $expectedBootstrapHash = ((Get-Content -LiteralPath $bootstrapChecksum -Raw).Trim() -split '\s+')[0].ToLowerInvariant()
    $actualBootstrapHash = (Get-FileHash -LiteralPath $bootstrapZip -Algorithm SHA256).Hash.ToLowerInvariant()
    if ($actualBootstrapHash -ne $expectedBootstrapHash) { throw "Bootstrap ZIP checksum mismatch; the download will not be used." }
    $stagingDirectory = "$packageDirectory.extracting"
    if (Test-Path -LiteralPath $stagingDirectory) { Remove-Item -LiteralPath $stagingDirectory -Recurse -Force }
    Expand-Archive -LiteralPath $bootstrapZip -DestinationPath $stagingDirectory -ErrorAction Stop
    if (Test-Path -LiteralPath $packageDirectory) { Remove-Item -LiteralPath $packageDirectory -Recurse -Force -ErrorAction Stop }
    Move-Item -LiteralPath $stagingDirectory -Destination $packageDirectory -ErrorAction Stop
    break
  } catch {
    Remove-Item -LiteralPath "$bootstrapZip.part" -Force -ErrorAction SilentlyContinue
    Write-Warning "Download/extraction attempt $attempt failed: $($_.Exception.Message)"
    if ($attempt -eq 3) { throw "Bootstrap was not prepared after three attempts. Correct the network or obtain a fresh package, then rerun Step 2." }
    Read-Host "Correct the reported problem, then press Enter to retry"
  }
}
Set-Location $packageDirectory
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Get-ChildItem -LiteralPath $packageDirectory -Recurse -File | Unblock-File
Get-ChildItem .\preflight-connectivity.ps1, .\verify-release.ps1, `
  .\configure-postgresql18.ps1, .\bootstrap-windows-server.ps1
Get-ChildItem .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe, `
  .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe.sha256
```

The process-scoped execution policy applies only to this PowerShell window; it does not change the
computer or user policy. `Unblock-File` removes the downloaded-file marker from the already
checksum-verified extracted copy. The commands must then list all four scripts plus one employee
installer and its checksum. Run every remaining command from that extracted package directory and
the same elevated PowerShell window.

## Step 3: Confirm you have the right key

Everything below trusts one file: `release-signing-public.pem`, included in this package. Before
using it, confirm its SHA-256 fingerprint matches the one the vendor gave you **through a separate
channel** from wherever you got this package — a phone call, a signed email, a contract appendix.
Never accept the fingerprint printed alongside the download itself; that's exactly what an attacker
controlling the download would also control.

```powershell
Get-FileHash .\release-signing-public.pem -Algorithm SHA256
```

If it doesn't match what the vendor told you, stop. Don't run anything else in this package, and
contact the vendor immediately.

## Step 4: Check the machine

`preflight-connectivity.ps1` confirms the machine is ready before you commit to an install. Its
checks are read-only and report every problem in one pass. The optional remediation mode below only
makes a change if you explicitly accept its pinned Node.js installation prompt.

The distribution copy of this guide fills in the exact manifest URL and signing-key fingerprint
when a release is published. Keep both values quoted if you copy this template from a source
checkout: PowerShell treats an unquoted `<placeholder>` as an operator.

```powershell
$manifestUri = "https://raw.githubusercontent.com/andrelch/term-sheet-extractor-dist/main/production.json"
$expectedFingerprint = "54ce5bf97695f05fa2223e6e8320d4b91445513e7210028863136e8faa833217"

.\preflight-connectivity.ps1 -ManifestUri $manifestUri `
  -PublicKeyPath .\release-signing-public.pem -ExpectedPublicKeyFingerprint $expectedFingerprint `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe -OfferRemediation
```

| Check | A failure means |
| --- | --- |
| DNS resolution | This machine can't resolve the update server's hostname. Check DNS configuration. |
| Outbound HTTPS to manifest host | A firewall or proxy is blocking outbound HTTPS. The server needs this reachable permanently, not just during install — it's how it gets every future update. |
| Release public key present / fingerprint match | Either the key file is missing, or it doesn't match the fingerprint you were given. **Treat a mismatch as a possible tampering attempt, not a typo** — stop and contact the vendor. |
| Node.js available | Node isn't installed, isn't on the expected path, is older than version 22, or isn't the x64 build. |
| System tar.exe available | Windows' built-in `tar.exe` (System32) is missing or shadowed by another `tar` on PATH. |
| NSSM / Caddy present | The path you gave doesn't point at a real file. |
| Elevated PowerShell | Step 7 needs an administrator window to configure services, ACLs, firewall rules, and scheduled tasks. |
| PostgreSQL 18 tools / service | The PostgreSQL server or required `psql`, `pg_dump`, and `pg_restore` tools are missing or the wrong major version. |
| Data volume BitLocker | `C:\TermSheetData` would be stored on a volume that is not yet fully encrypted and protected. Enable BitLocker and escrow its recovery key before Step 7. |

Fix everything reported before continuing — each one will surface again, less clearly, during actual
install or first update.

The JSON result includes a `remediation` for every failed check. With `-OfferRemediation`, the script
also prints those actions and can install the pinned Node.js 24 x64 package after confirmation when
WinGet is present. Network, tool-path, and Windows-component repairs are proposed without changing
company policy. A missing key can be repaired by re-extracting a clean package; a fingerprint
mismatch is never bypassed and must be escalated to the vendor. Fix the listed items and rerun the
same command until every check passes.

## Step 5: Verify the release before installing

`verify-release.ps1` downloads the exact release your key would install, checks its signature,
size, and checksum, and confirms its release marker — the same checks the installer and every future
update apply — then deletes the download. It changes nothing on the machine.

```powershell
.\verify-release.ps1 -ManifestUri $manifestUri -PublicKeyPath .\release-signing-public.pem
```

A clean run prints the version and release identifier it verified. Any failure here means don't
proceed with install — something about the release doesn't check out, and continuing anyway would
just fail the same check again during installation, or worse, not fail it.

For a timeout or HTTPS error, apply the matching Step 4 network remediation and rerun verification.
For a missing file, re-extract the verified bootstrap ZIP into a fresh directory. A signature,
checksum, release-marker, or signing-key failure is not repairable on the client machine: discard
the downloaded copy, obtain a fresh package, and contact the vendor if the fresh copy also fails.
There is no supported switch that bypasses these checks.

## Step 6: Understand the automatic server configuration

There is no environment-file step to perform. During Step 7, the bootstrap automatically runs the
verified `configure-postgresql18.ps1` helper, prompts securely for the PostgreSQL administrator and
application-role passwords it needs, and receives the application password as a `SecureString`.
It then creates `%ProgramData%\WinnerZone\TermSheet\config\server.env` from the packaged baseline
and fills in the escaped `DATABASE_URL`, a new 48-byte `AUTH_SECRET`, `C:\TermSheetData\objects`,
`APP_MODE=dual`, the selected DPAPI or Key Vault provider, and local installation-owner access. The
local workspace door is accepted only over loopback. It never prints a
password or writes a temporary password file.

The helper safely handles fresh installs, reruns, existing roles, and legacy or third-party database
owners. Empty, mismatched, or rejected passwords prompt again with a specific recovery action.
For a PostgreSQL owner created with `NOLOGIN`, the authenticated `postgres` administrator must type `TRANSFER` explicitly before ownership changes.
On every rerun, the bootstrap reconciles database grants and ownership before Prisma migrations and
refreshes the managed local `DATABASE_URL` with the verified application-role credential. It
preserves the existing authentication secret and other operator configuration. It also rejects a
populated `DOCUMENT_STORE_MASTER_KEY`, because the central key belongs in DPAPI or Azure Key Vault
rather than in an environment file.

Provider API keys deliberately remain blank. After installation, use the localhost server launcher,
choose **Developer access**, and enter approved keys under **Settings -> API keys**. The default
`SECRET_STORE=encrypted` stores them through the application's encrypted secret store. Staff and
provider configuration therefore require no Notepad session and no direct access to `server.env`.

## Step 7: Install

From an elevated PowerShell session, in this package's directory, run one complete block below. Each
block obtains the identity and credential again, so it also works in a newly elevated window. On a fresh
installation it calls the packaged PostgreSQL helper and prompts for the necessary database
passwords before downloading the release. It then creates the protected server configuration
automatically; do not create or edit `server.env` first.

The installer prints the phase and total elapsed time whenever it begins work or is still waiting.
It also records the complete session in
`%ProgramData%\WinnerZone\TermSheet\installation-logs\bootstrap-<date>-<time>.log`. If installation
stops, the last screen clearly identifies the failed phase, the exact error, the script location,
the transcript path, and the service-log directory. Completed phases are retained, so leave the
window open, correct the reported condition, and rerun the same block. Do not delete an installation
directory to retry. When requesting support, send the newest installation transcript and only the
named `.err.log`; neither file should contain the password entered into the secure credential dialog.

With two-custodian escrow:

Before running this block, copy the two custodians' distinct public `.cer` files to the exact paths
shown (or replace both paths in the command). The private keys remain with the custodians.

```powershell
$servicePolicy = Get-ExecutionPolicy -Scope Process
if ($servicePolicy -ne "Bypass") { Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force }
Get-ChildItem -LiteralPath $PWD -Recurse -File | Unblock-File

$serviceUser = [Security.Principal.WindowsIdentity]::GetCurrent().Name
do {
  try {
    $serviceSid = (New-Object Security.Principal.NTAccount($serviceUser)).Translate([Security.Principal.SecurityIdentifier]).Value
    $serviceCredential = Get-Credential -UserName $serviceUser `
      -Message "Log in to the Windows account that will run the Term Sheet services"
    if (-not $serviceCredential) { Write-Warning "No credential was entered." }
  } catch {
    Write-Warning "'$serviceUser' did not resolve: $($_.Exception.Message)"
    $serviceUser = Read-Host "Enter COMPUTERNAME\username or DOMAIN\username, or press Enter to stop"
    if (-not $serviceUser) { return }
  }
} until ($serviceSid -and $serviceCredential)

do {
  $publicHostname = Read-Host "Enter the employee DNS hostname, or press Enter for termsheetextractor.local"
  if (-not $publicHostname) { $publicHostname = "termsheetextractor.local" }
  $hostnameIsValid = $publicHostname -match '^[A-Za-z0-9](?:[A-Za-z0-9.-]*[A-Za-z0-9])?$' -and -not $publicHostname.Contains('..')
  if (-not $hostnameIsValid) { Write-Warning "That is not a valid DNS hostname. Try again or press Enter for termsheetextractor.local." }
} until ($hostnameIsValid)

.\bootstrap-windows-server.ps1 -ApplicationRoot C:\TermSheet `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe `
  -PublicHostname $publicHostname `
  -ServiceUser $serviceUser -ServiceCredential $serviceCredential `
  -EscrowDirectory C:\TermSheetKeyEscrow -EscrowCertificatePath C:\TermSheetKeyEscrow\custodian-a.cer,C:\TermSheetKeyEscrow\custodian-b.cer `
  -ManifestUri $manifestUri -ReleasePublicKeyPath .\release-signing-public.pem
```

If no custodians have been appointed, the client may formally accept the temporary recovery risk
and omit both escrow arguments by using `-AllowDeferredEscrow`:

```powershell
$servicePolicy = Get-ExecutionPolicy -Scope Process
if ($servicePolicy -ne "Bypass") { Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force }
Get-ChildItem -LiteralPath $PWD -Recurse -File | Unblock-File

$serviceUser = [Security.Principal.WindowsIdentity]::GetCurrent().Name
do {
  try {
    $serviceSid = (New-Object Security.Principal.NTAccount($serviceUser)).Translate([Security.Principal.SecurityIdentifier]).Value
    $serviceCredential = Get-Credential -UserName $serviceUser `
      -Message "Log in to the Windows account that will run the Term Sheet services"
    if (-not $serviceCredential) { Write-Warning "No credential was entered." }
  } catch {
    Write-Warning "'$serviceUser' did not resolve: $($_.Exception.Message)"
    $serviceUser = Read-Host "Enter COMPUTERNAME\username or DOMAIN\username, or press Enter to stop"
    if (-not $serviceUser) { return }
  }
} until ($serviceSid -and $serviceCredential)

do {
  $publicHostname = Read-Host "Enter the employee DNS hostname, or press Enter for termsheetextractor.local"
  if (-not $publicHostname) { $publicHostname = "termsheetextractor.local" }
  $hostnameIsValid = $publicHostname -match '^[A-Za-z0-9](?:[A-Za-z0-9.-]*[A-Za-z0-9])?$' -and -not $publicHostname.Contains('..')
  if (-not $hostnameIsValid) { Write-Warning "That is not a valid DNS hostname. Try again or press Enter for termsheetextractor.local." }
} until ($hostnameIsValid)

.\bootstrap-windows-server.ps1 -ApplicationRoot C:\TermSheet `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe `
  -PublicHostname $publicHostname `
  -ServiceUser $serviceUser -ServiceCredential $serviceCredential `
  -AllowDeferredEscrow `
  -ManifestUri $manifestUri -ReleasePublicKeyPath .\release-signing-public.pem
```

This is not a recovery mechanism. Until escrow is added, losing the Windows machine or its key
provider can make every encrypted document permanently unreadable. Record who approved the
exception, protect system backups, and appoint custodians as soon as possible.

What each required flag is:

| Flag | What it is |
| --- | --- |
| `-ApplicationRoot` | Where releases are installed. Not where your data lives. |
| `-NssmPath`, `-CaddyPath` | Paths to the tools you installed in [Step 1](#step-1-before-you-start). |
| `-PublicHostname` | Defaults to `termsheetextractor.local`. Step 7 puts the current LAN address in the employee installer; setup replaces older mappings for that hostname on each employee computer automatically. Pass `localhost` explicitly for a server-only installation. Caddy also keeps `https://localhost/` available for the server-console launcher. |
| `-ClientFilesRoot` | Optional output directory for the verified combined employee installer and its handoff files. Defaults to `C:\TermSheet\Client Files`. |
| `-ServiceUser` | The Windows account the application services run as. These blocks default to the signed-in account; an approved dedicated low-privilege service identity may be substituted. |
| `-EscrowDirectory`, `-EscrowCertificatePath` | Where sealed, encrypted backups of the document-encryption master key are written, and the public certificates of two separate recovery holders. Each certificate produces an independently recoverable package. |
| `-AllowDeferredEscrow` | Explicitly installs without recovery packages when custodians do not yet exist. It cannot be combined with either escrow argument and must be treated as a temporary, approved data-loss risk. |
| `-ManifestUri` | The vendor's update-channel URL. Fixed — it's the same URL for every future update too. |
| `-ReleasePublicKeyPath` | The key from [Step 3](#step-3-confirm-you-have-the-right-key). |

It's safe to run more than once, including from a newly downloaded bootstrap package. Treat a newer
bootstrap as an authoritative maintenance installation: Step 7 does not return successfully until
the signed release it names is active and healthy against the existing external database, document
store, keys and configuration. Older release directories remain as rollback copies, so their
presence under `C:\TermSheet\releases` does not identify the running version; the `current` junction
and **Settings -> System** do.

A complete release at the channel version is left in place. If an older release was left behind by a bootstrap
that stopped before creating the server-key marker or any Term Sheet service, the rerun installs the
newer verified release and repoints only `current`; the old versioned release directory remains
preserved. On an initialized server, bootstrap keeps the active release serving only while it
refreshes the external updater, service definitions, shortcuts, and employee handoff. It then waits
synchronously for the refreshed signed updater to download, migrate, health-check and activate the
newer release; a queued, failed or still-old result makes Step 7 fail rather than print installation
success. Existing data, secrets, configuration, signing
trust, and rollback releases remain intact. Bootstrap refuses to trust a different signing key on a
rerun — key rotation is a separate, deliberate procedure the vendor will walk you through.

If bootstrap stops, use the red **INSTALLATION STOPPED** summary rather than searching backward
through the console. Keep its error message and transcript, apply the specific fix it proposes, and
rerun the same Step 7 command. It resumes from verified state and recreates missing services or launcher files; do
not delete `C:\TermSheet`, the data directory, or `%ProgramData%\WinnerZone\TermSheet` to retry.
The updater stage is transaction-safe and is attempted up to three times before bootstrap stops.
Each attempt verifies both enabled scheduled tasks and a readable versioned state file, so a transient
ACL or Task Scheduler failure cannot be mistaken for a completed installation.
After services and launchers are ready, Step 7 starts both tasks under their installed `SYSTEM`
identity. It waits for a successful fresh channel check (and any offered signed update), verifies the
task exit results and updater state, then checks server readiness again. A large first update can keep
Step 7 open while it downloads, backs up, and validates the release; do not close that window.
Immediately before that maintenance update, Step 7 quiesces the extraction and backup workers. It
uses the official Windows service-stop path and waits for NSSM to finish the registered shutdown
sequence without imposing an additional installer timeout or killing worker processes. The workers
are restarted after activation or restored if the installation fails, so background work cannot
outrank installation and is not left disabled.

When the two public certificates become available, add escrow to the existing key from an elevated
PowerShell session. This command does not rotate the key or rewrite encrypted documents:

```powershell
.\add-server-key-escrow.ps1 `
  -EscrowDirectory E:\TermSheetKeyEscrow\initial `
  -EscrowCertificatePath C:\KeyBootstrap\custodian-a.cer,C:\KeyBootstrap\custodian-b.cer
```

Give each recovery holder their matching `.p7m` package, verify its SHA-256 value against
`%ProgramData%\WinnerZone\TermSheet\installation-key.json`, and remove the temporary public
certificate copies from the server.

## Step 8: Confirm the installation

Four Windows services, all managed by NSSM, all set to restart automatically:

| Service | What it does |
| --- | --- |
| `TermSheetWeb` | The application itself |
| `TermSheetWorker` | Background processing (document extraction, etc.) |
| `TermSheetProxy` | Caddy — terminates HTTPS on port 443 |
| `TermSheetBackup` | Scheduled database/document backups |

Logs for each are at `%ProgramData%\WinnerZone\TermSheet\logs\<ServiceName>.log` (and `.err.log` for
errors). From the same verified package directory and an elevated PowerShell window, run this entire
block. It avoids cascading command errors when a component is absent and does not report success
unless every required check passes:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Get-ChildItem -LiteralPath $PWD -Recurse -File | Unblock-File
.\repair-windows-server.ps1 -AcceptSafeRepairs
if ($LASTEXITCODE -ne 0) {
  Write-Warning "Step 8 found a problem. Apply every recovery action printed above, then rerun this block."
  return
}
Write-Host "Step 8 server checks passed."
```

The helper checks elevation; the PostgreSQL and four application services and their automatic-start
configuration; both scheduled updater tasks; updater JSON and active-release version agreement;
ports `443`, `3000`, and `5432`; loopback-only binding for internal ports; web readiness; the worker
heartbeat; logs; both installed server shortcuts; and every employee handoff file. It also validates
the employee installer checksum, server URL, shortcut target, and the complete ZIP contents. Brief
retries cover ordinary service startup timing. The command automatically restores automatic startup,
starts stopped services, enables scheduled tasks, runs the updater when its state is missing, and
restores user shortcuts. If a required port, web readiness check, or worker heartbeat is unhealthy,
it restarts the responsible service once and verifies it again. It never silently rebuilds or replaces
missing application or customer data.
The audited `Client Files` set is `Term-Sheet-Extractor-Employee-Setup.exe`, its `.sha256` file,
`employee-launcher.json` (including the independently versioned launcher release),
`Term Sheet Extractor - Server.url`, `server-url.txt`, `README.txt`, and
`Term-Sheet-Client-Files.zip`.

For ordinary version checks after Step 8 passes, use **Settings → System → Server version and
updates**. If the helper reports a problem, follow its exact recovery action and inspect the named
`.err.log`; do not continue to owner setup until the helper exits successfully.

The bootstrap has already installed `Term Sheet Extractor - Server` in the current and all-users
Desktop and Start menu locations, so it remains visible even when Step 7 ran elevated. Open it now
and confirm the sign-in page loads. This is the supported way to
launch the UI directly from the server computer, including a localhost-only installation.

On that localhost sign-in page, choose **Developer access**. This provisions and signs in the sole
installation owner through the server's loopback-only owner door. In **Settings → Members**, create
an active normal administrator and verify that person can sign in through the configured staff
method before removing owner access. Then, from an elevated PowerShell session on the server, run:

```powershell
& "C:\TermSheet\current\deploy\set-local-owner-access.ps1" -Enabled false
```

The command changes only `DEBUG_AUTH` in the protected server configuration and restarts the web
service, invalidating the debug-owner session. It must be run by a Windows server administrator;
the application administrator should first confirm their normal sign-in works. To recover the owner
door later, use the same command with `-Enabled true`.

In **Settings**, enter approved provider credentials under **API keys**. The default encrypted secret
store persists those keys without exposing or editing `server.env`. Employee computers cannot use
Developer access; they use the staff authentication configured by the owner.

## Step 9: Verify the mandatory combined employee installer

`C:\TermSheet\Client Files\Term-Sheet-Client-Files.zip` is the employee handoff. It contains one
Tauri NSIS executable—`Term-Sheet-Extractor-Employee-Setup.exe`—with both employee choices:

- **Sign in to the office server** opens the shared HTTPS application. The server owns the central
  database, documents, workers, approvals, backups, and updater.
- **Work on this computer** starts the bundled Node application and private PostgreSQL on loopback.
  It belongs to one Windows user, uses the Windows credential store for its document key, and has no
  application sign-in.

The employee performs no PostgreSQL steps. They do not download PostgreSQL, run an installer for it,
choose a database password, create a database or Windows service, select a port, edit configuration,
or run a command. On the first **Work on this computer** click, the launcher initializes its bundled
private cluster, creates the application database, applies the bundled schema in one transaction,
supplies a generated SCRAM database credential, and opens the UI. Later launches start that same
private cluster automatically; closing the launcher stops it. A release is blocked unless this exact
automatic initialization passes against the bundled binaries and schema.

Before distribution, compare the executable with
`Term-Sheet-Extractor-Employee-Setup.exe.sha256`. Keep `server-url.txt` beside the installer when
running setup: the installer copies this installation-specific server address beside the installed
executable automatically. IT can deploy the same file there later when retargeting is required. Do
not copy `server.env`, `ProgramData\WinnerZone\TermSheet\config`, or any server secret to an employee
computer.

Local mode is intentionally separate from the company's book. It does not synchronize, cannot make
central maker/checker approvals, and uses self-review labels. To move work into the company book,
re-upload the original document to server mode. Local extraction also requires an approved,
separately provisioned provider key. Losing the Windows profile loses the credential-store key and
therefore the local documents, so managed endpoints still require BitLocker and normal device backup
policy.

The central server stack is installed only on the server computer. Employee computers install this
single launcher/local-workspace package; they do not install the server's NSSM services, proxy,
updater, or shared database. The release workstation builds and signs the installer but does not run
another copy of the client's central server.

If `PublicHostname` is `localhost`, server mode is usable only on the server computer; `localhost` on
an employee computer means that employee computer. Local mode still works there. For employee access
to the shared server, configure a DNS hostname reachable from those computers. If Caddy uses its
internal CA, IT must also deploy that trust certificate through its normal Windows policy.

## Day-to-day operation

**Updates are automatic and require nothing from you.** The updater checks the manifest URL every
15 minutes by default (configurable at install time), and applies a new signed version if one's
available — downloading it, verifying its signature, taking a backup, running it alongside the
current version to confirm it's healthy, and only then switching over. If a newly-installed version
fails its own health check, the updater automatically switches back to the previous one; you don't
have to intervene, and the change happens in seconds.

To check status at any time, sign in with the **Operate the server** capability and open
**Settings → System → Server version and updates**. It shows the running and available versions,
the last check and successful installation times, failures, and any safe rollback target. When the
rollback guard permits it, provide a reason and queue the rollback there; the server restarts and
the screen reconnects automatically.

An installation created with an older bootstrap may show **Install the current bootstrap maintenance
package once to enable interface rollback**. Rerun the current verified bootstrap package; it keeps
the installed application, configuration, signing key, and data in place while refreshing the
out-of-release updater and adding the one-minute rollback processor.

The version in the downloaded `term-sheet-bootstrap-<version>` folder is only the version of that
installation/repair package. Its name never changes after installation and must not be used to
identify the running server version.

If the interface is unavailable, use this PowerShell fallback:

```powershell
$stateFile = Join-Path $env:ProgramData "WinnerZone\TermSheet\updater-state.json"
Get-Content -LiteralPath $stateFile -Raw | ConvertFrom-Json
```

Use `$env:ProgramData` in PowerShell. `%ProgramData%` is Command Prompt syntax; PowerShell can treat
it as a relative directory beneath the current VS Code or `dist` folder.

| Field | Meaning |
| --- | --- |
| `status` | What the updater is doing right now: `idle` (nothing to do), `checking`, `downloading`, `staging`, `snapshotting` (taking a pre-update backup), `activating`, `rolling-back`, or `failed`. |
| `installedVersion` | The version currently serving traffic. |
| `previousVersion` | What was running before the last successful update — the rollback target if needed. |
| `blockedVersion` | A version the updater tried and rejected (failed its health check, or a rollback failed). It will not retry this exact version again; a newer signed release is required. If this is set, contact the vendor — this generally means a release needs a fix, not that anything is wrong with your server. |
| `activeReleaseUnhealthy` | If `true`, the currently active release failed its own health checks and an automatic rollback wasn't possible (most often because a database migration can't safely run backwards). This is the one status worth paging someone over — the server may be degraded and needs the vendor's attention, ideally restored from the backup noted in `lastSuccessfulUpdateAt`. |
| `lastCheckedAt`, `lastSuccessfulUpdateAt`, `lastFailureAt` | Timestamps, for confirming the updater is actually running on schedule. |

If you gave `-UpdateStatusWebhookUri` at install time, every status change is also posted there
automatically — useful for wiring into existing monitoring rather than polling the file above.

### Starting the server and opening the UI again

All Term Sheet services are configured for automatic startup. After a normal Windows restart, sign
in to the server computer with the Windows account used during setup, wait for the services to start,
then open **Term Sheet Extractor - Server** from the Desktop or Start menu.

To start or verify everything manually, open **Windows PowerShell as Administrator** in the retained
bootstrap directory and run. Add `-AcceptSafeRepairs` to start stopped services and the updater
without individual prompts; missing components are never guessed or installed from an untrusted
source:

```powershell
.\repair-windows-server.ps1

$localLauncher = "C:\TermSheet\Client Files\Term Sheet Extractor - Server.url"
if (Test-Path -LiteralPath $localLauncher) { Start-Process $localLauncher }
```

If PostgreSQL 18 is missing, reinstall PostgreSQL 18 and rerun Step 7. If a Term Sheet
service, updater task, or launcher file is missing, rerun Step 7 with the same values. Bootstrap
preserves the pinned signing key, installed release, data, and existing authentication secret.

### Stopping the server for NSSM or tool maintenance

An idle `TermSheetBackup` does not imply a backup is running. Older installations gave its
PowerShell wrapper a 30-minute NSSM console-stop timeout, which can leave the service in
`STOP_PENDING` while it prints output. Current releases handle NSSM's Windows `CTRL+BREAK` signal
directly so idle processes exit promptly, and current Step 7 also reduces the forced-stop fallback
to one minute. To stop all NSSM-hosted Term Sheet services and prevent the updater from restarting
them, use elevated PowerShell:

```powershell
$taskNames = "WinnerZone Term Sheet Updater", "WinnerZone Term Sheet Rollback Processor"
foreach ($taskName in $taskNames) {
  Disable-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue | Out-Null
  Stop-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
}

$serviceNames = "TermSheetBackup", "TermSheetWorker", "TermSheetWeb", "TermSheetProxy"
foreach ($serviceName in $serviceNames) { sc.exe stop $serviceName | Out-Null }
$deadline = (Get-Date).AddSeconds(90)
do {
  $remaining = @(Get-Service -Name $serviceNames -ErrorAction SilentlyContinue |
    Where-Object Status -ne "Stopped")
  if (-not $remaining.Count) { break }
  Start-Sleep -Seconds 3
} while ((Get-Date) -lt $deadline)

# Only the Term Sheet NSSM service trees still stuck after the graceful request are terminated.
foreach ($service in $remaining) {
  $serviceProcess = Get-CimInstance Win32_Service -Filter "Name='$($service.Name)'"
  if ($serviceProcess.ProcessId -gt 0) {
    taskkill.exe /PID $serviceProcess.ProcessId /T /F | Out-Null
  }
}
Get-Service -Name $serviceNames
```

All four must show `Stopped` before replacing `C:\tools\nssm.exe`. PostgreSQL is deliberately not
stopped. After maintenance, rerun Step 7 (preferred, because it reapplies current service settings),
or start the four services and re-enable both scheduled tasks.

If the browser does not load, confirm the application answers on loopback before debugging DNS or
certificates:

```powershell
Invoke-RestMethod http://127.0.0.1:3000/api/health/live
Get-Content "$env:ProgramData\WinnerZone\TermSheet\logs\TermSheetWeb.err.log" -Tail 50
Get-Content "$env:ProgramData\WinnerZone\TermSheet\logs\TermSheetProxy.err.log" -Tail 50
```

## If something goes wrong

**A service won't start.** Check its `.err.log` under `logs\`. Confirm PostgreSQL is reachable and
`server.env` is present and correctly restricted (only the service account and administrators should
be able to read it — the bootstrap sets this automatically, but a later manual edit can undo it).

**A previous bootstrap stopped at “Database migration failed” and an administrator created a Prisma
config to recover it.** Leave that file in place and rerun the current bootstrap. It prefers the
signed release's `prisma.config.mjs`, but when resuming an older partial installation it also accepts
Prisma-supported `prisma.config.*` or `.config\prisma.*` files under `C:\TermSheet\current` or
`C:\TermSheet`. The bootstrap prints the compatibility file it selected. A later successful signed
release supplies its own config, so do not copy the client-created file into new release directories.

**The employee launcher opens the wrong hostname.** The compiled default can be overridden with
`server-url.txt`. Confirm the generated file contains the correct HTTPS URL and that setup was run
with that file beside the installer. Setup copies it beside the installed executable automatically;
the launcher distribution mechanism can also place or replace it there later:

```powershell
Get-Content -LiteralPath "C:\TermSheet\Client Files\server-url.txt" -Raw
```

**`preflight-connectivity.ps1` or `verify-release.ps1` starts failing after months of working fine.**
Most likely an outbound firewall or proxy rule changed. If it's specifically the public-key
fingerprint check that starts failing, stop and contact the vendor before doing anything else — that
specific failure is what a compromised update channel would look like.

**Disk space errors during an update.** The updater refuses to proceed unless there's room for the
new release plus enough headroom to roll back safely. Free up space; nothing will be left half
-installed by a refusal like this.

**You need to force an update check immediately**, rather than waiting for the next scheduled run.
From elevated PowerShell, run the installed scheduled task; this command is independent of the
current directory:

```powershell
Start-ScheduledTask -TaskName "WinnerZone Term Sheet Updater"
```

Then read `$stateFile` as shown above. If that file does not exist, bootstrap did not complete or the
installation used a custom `StateRoot`; inspect the scheduled task action for its configured path:

```powershell
(Get-ScheduledTask -TaskName "WinnerZone Term Sheet Updater").Actions
```

## Getting help

Contact your vendor with the version from `updater-state.json`'s `installedVersion`, the relevant
lines from the service's `.err.log`, and — if it's update-related — the full `updater-state.json`.
Don't send `server.env` or anything from the `config\` or `secrets\` folders; the vendor should never
need those, and they contain your credentials.
