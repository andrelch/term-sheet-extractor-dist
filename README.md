# Term Sheet server — operator guide

> Verified against the current bootstrap, updater, release layout, and service installer on
> 27 August 2026.

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
whether two custodian certificates are currently available, BitLocker recovery-key custody, and
approved provider API keys. Node.js 24, PostgreSQL 18, NSSM, and Caddy are machine prerequisites;
Step 1 provides verified installation commands because enterprise package policy and the PostgreSQL
administrator password cannot safely be invented by the application. Numbered steps identify
operator interaction points; automatic checks and explanatory sections are not separate steps.

Once installed, Windows starts the server automatically after reboot. The installation and repair
commands appear only after the guide has downloaded the scripts that provide them. Later, the
[day-to-day operation](#day-to-day-operation) section covers starting or repairing the server on
demand.

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
organization has not appointed them yet, Step 3 asks whether to install with or without them and
provides an explicit temporary deferred-escrow
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

If PostgreSQL 18 is already present, keep it and continue; the Step 3 bootstrap helper discovers and
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
  throw "C: must be fully encrypted and protected before installation."
}
```

Database configuration happens with the verified `configure-postgresql18.ps1` helper during
Step 3. Do not manually create roles or a database here. The helper handles fresh installations,
safe reruns, existing login or non-login roles, forgotten application-role passwords, a database
already owned by `term_sheet_extractor_admin`, the older `term_sheet_app` ownership layout, and a
database owned by another administrator. It also corrects runtime grants, the firewall rule, service
startup, loopback binding, and ownership of pre-existing application objects in the dedicated
`public` schema. Password mistakes prompt again instead of terminating setup.

### Step 1.3: Install the remaining prerequisites

Obtain and extract the verified bootstrap ZIP first (the package preparation block in Step 2
can also reuse a ZIP extracted on another computer). Run the prerequisite helper from that folder
in elevated Windows PowerShell. Do not run machine preflight until these prerequisites are installed.

```powershell
.\install-windows-prerequisites.ps1
```

This installs the modern-Windows-compatible NSSM 2.24-101 x64 build and the latest stable Caddy
x64 build into `C:\tools`. Both archives are verified before any service is stopped: NSSM uses
its published SHA-1 digest and Caddy uses its release-specific SHA-512 checksum. Both executables
must pass a version check before replacing existing tools. Running services sharing this NSSM
copy are briefly stopped and restarted; schedule a maintenance window if other applications use it.
Old binaries and staging files are retained at the printed recovery path.

Online requests have a 60-second timeout and at most three attempts. If `api.github.com` is blocked
but GitHub release downloads work, pass an organization-approved stable version explicitly:

```powershell
$caddyVersion = Read-Host "Enter the approved Caddy version (x.y.z)"
.\install-windows-prerequisites.ps1 -CaddyVersion $caddyVersion
```

If the server has no approved GitHub access, download the following on a connected, trusted computer
and transfer them together into a dedicated folder on an existing drive:

- `nssm-2.24-101-g897c7ad.zip` from the [official NSSM download page](https://nssm.cc/download).
- `caddy_VERSION_windows_amd64.zip` and `caddy_VERSION_checksums.txt` from the same approved
  [official Caddy release](https://github.com/caddyserver/caddy/releases).

Authenticate the checksum file through your organization's trusted download/transfer process;
a checksum copied alongside an untrusted archive does not establish authenticity.

```powershell
$offlineDirectory = Read-Host "Enter the absolute folder containing the approved archives and checksums"
$caddyVersion = Read-Host "Enter the matching Caddy version (x.y.z)"
.\install-windows-prerequisites.ps1 -OfflineDirectory $offlineDirectory -CaddyVersion $caddyVersion
```

Offline mode makes no network requests and applies the same checksum/version checks. It never
falls back to an unverified executable. Missing files, invalid versions and checksum mismatches
stop before replacing tools. Correct DNS/proxy/firewall policy with the network administrator;
do not disable TLS validation or replace the server's DNS settings blindly. This offline option
covers NSSM/Caddy only; the later signed application-release workflow still needs its documented access.

NSSM stable 2.24 is not supported: its version command exits with code 1 on modern Windows.
The service identity and credential are collected once, immediately before installation in Step 3.

## Step 2: Prepare and verify the bootstrap package

The distribution repository root intentionally contains only the update channel, signing key, and
this guide. The PowerShell scripts are in the versioned bootstrap ZIP attached to the current server
release. Run the following block once. If the scripts are already beside this guide, it reuses that
extracted package. Otherwise it downloads, checksum-verifies, and extracts the package first. It
then authenticates the signing key, checks the machine, and verifies the signed release without
making you switch between separate command blocks.

```powershell
$bootstrapUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.3.18/term-sheet-bootstrap-0.3.18.zip"
$bootstrapChecksumUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.3.18/term-sheet-bootstrap-0.3.18.zip.sha256"
$manifestUri = "https://raw.githubusercontent.com/andrelch/term-sheet-extractor-dist/main/production.json"
$publishedFingerprint = "54ce5bf97695f05fa2223e6e8320d4b91445513e7210028863136e8faa833217".ToLowerInvariant()

if (Test-Path -LiteralPath (Join-Path $PWD "preflight-connectivity.ps1") -PathType Leaf) {
  $packageDirectory = $PWD.Path
} else {
  $downloadRoot = Join-Path $PWD "term-sheet-bootstrap-0.3.18-download"
  $bootstrapZip = Join-Path $downloadRoot "term-sheet-bootstrap-0.3.18.zip"
  $bootstrapChecksum = "$bootstrapZip.sha256"
  $packageDirectory = Join-Path $downloadRoot "term-sheet-bootstrap-0.3.18"
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
}
Set-Location $packageDirectory
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Get-ChildItem -LiteralPath $packageDirectory -Recurse -File | Unblock-File

$requiredFiles = @(
  "preflight-connectivity.ps1",
  "verify-release.ps1",
  "configure-postgresql18.ps1",
  "bootstrap-windows-server.ps1",
  "repair-windows-server.ps1",
  "export-installation-support-bundle.ps1",
  "release-signing-public.pem"
)
foreach ($requiredFile in $requiredFiles) {
  if (-not (Test-Path -LiteralPath (Join-Path $PWD $requiredFile) -PathType Leaf)) {
    throw "The extracted bootstrap package is missing $requiredFile."
  }
}
$employeeInstaller = @(Get-ChildItem `
  .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe -File)
$employeeChecksum = @(Get-ChildItem `
  .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe.sha256 -File)
if ($employeeInstaller.Count -ne 1 -or $employeeChecksum.Count -ne 1) {
  throw "The package must contain exactly one employee installer and its checksum."
}

$expectedFingerprint = $publishedFingerprint
if ($expectedFingerprint -notmatch '^[0-9a-f]{64}$') {
  throw "This published guide does not contain a valid pinned signing-key fingerprint. Obtain a fresh release copy."
}
$actualFingerprint = (
  Get-FileHash .\release-signing-public.pem -Algorithm SHA256
).Hash.ToLowerInvariant()
if ($expectedFingerprint -ne $actualFingerprint) {
  throw "The signing-key fingerprint does not match. Stop and contact the vendor."
}

$global:LASTEXITCODE = 0
.\preflight-connectivity.ps1 -ManifestUri $manifestUri `
  -PublicKeyPath .\release-signing-public.pem `
  -ExpectedPublicKeyFingerprint $expectedFingerprint `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe -OfferRemediation
if ($LASTEXITCODE -ne 0) {
  throw "Machine preflight failed. Apply every reported remediation and rerun Step 2."
}

$global:LASTEXITCODE = 0
.\verify-release.ps1 -ManifestUri $manifestUri `
  -PublicKeyPath .\release-signing-public.pem `
  -ExpectedPublicKeyFingerprint $expectedFingerprint
if ($LASTEXITCODE -ne 0) {
  throw "Signed release verification failed. Do not continue to installation."
}
Write-Host "Step 2 passed. Keep this window open and continue to Step 3."
```

The process-scoped execution policy applies only to this PowerShell window; it does not change the
computer or user policy. `Unblock-File` removes the downloaded-file marker from the already
checksum-verified extracted copy. Run every remaining command from that extracted package directory
and the same elevated PowerShell window.

### Automatic signing-key authentication

The combined block trusts one file: `release-signing-public.pem`, included in this package. Release
publication pins that file's SHA-256 fingerprint into this guide automatically. The block computes
the downloaded key's fingerprint and compares it with the pinned value before it runs any packaged
script. The operator is never asked to find, copy, or retype a hash. If the values do not match, stop,
discard the download, and contact the vendor immediately.

### Machine checks

`preflight-connectivity.ps1` confirms the machine is ready before you commit to an install. Its
checks are read-only and report every problem in one pass. The optional remediation mode below only
makes a change if you explicitly accept its pinned Node.js installation prompt.

The distribution copy of this guide fills in the exact manifest URL and pinned signing-key
fingerprint when a release is built, then the block compares the pinned and computed values before
running the preflight.

| Check | A failure means |
| --- | --- |
| DNS resolution | This machine can't resolve the update server's hostname. Check DNS configuration. |
| Outbound HTTPS to manifest host | A firewall or proxy is blocking outbound HTTPS. The server needs this reachable permanently, not just during install — it's how it gets every future update. |
| Release public key present / fingerprint match | Either the key file is missing, or it doesn't match the fingerprint pinned at release publication. **Treat a mismatch as a possible tampering attempt, not a typo** — stop and contact the vendor. |
| Node.js available | Node isn't installed, isn't on the expected path, is older than version 22, or isn't the x64 build. |
| System tar.exe available | Windows' built-in `tar.exe` (System32) is missing or shadowed by another `tar` on PATH. |
| NSSM / Caddy present | The path you gave doesn't point at a real file. |
| Elevated PowerShell | Step 3 needs an administrator window to configure services, ACLs, firewall rules, and scheduled tasks. |
| PostgreSQL 18 tools / service | The PostgreSQL server or required `psql`, `pg_dump`, and `pg_restore` tools are missing or the wrong major version. |
| Data volume BitLocker | `C:\TermSheetData` would be stored on a volume that is not yet fully encrypted and protected. Enable BitLocker and escrow its recovery key before Step 3. |

Fix everything reported before continuing — each one will surface again, less clearly, during actual
install or first update. Rerun the complete Step 2 block after applying the remediation; it reuses
the extracted package rather than downloading it again.

The JSON result includes a `remediation` for every failed check. With `-OfferRemediation`, the script
also prints those actions and can install the pinned Node.js 24 x64 package after confirmation when
WinGet is present. Network, tool-path, and Windows-component repairs are proposed without changing
company policy. A missing key can be repaired by re-extracting a clean package; a fingerprint
mismatch is never bypassed and must be escalated to the vendor. Fix the listed items and rerun the
same command until every check passes.

### Signed-release verification

The combined block runs `verify-release.ps1`, which downloads the exact release your key would install, checks its signature,
size, and checksum, and confirms its release marker — the same checks the installer and every future
update apply — then deletes the download. It changes nothing on the machine.

A clean run prints the version and release identifier it verified. Any failure here means don't
proceed with install — something about the release doesn't check out, and continuing anyway would
just fail the same check again during installation, or worse, not fail it.

For a timeout or HTTPS error, apply the matching preflight network remediation and rerun the complete
Step 2 block.
For a missing file, re-extract the verified bootstrap ZIP into a fresh directory. A signature,
checksum, release-marker, or signing-key failure is not repairable on the client machine: discard
the downloaded copy, obtain a fresh package, and contact the vendor if the fresh copy also fails.
There is no supported switch that bypasses these checks.

## What installation configures automatically

There is no environment-file step to perform. During Step 3, the bootstrap automatically runs the
verified `.\configure-postgresql18.ps1` helper, prompts securely for the PostgreSQL administrator and
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

## Step 3: Answer the installation questions and install

From an elevated PowerShell session, in this package's directory, run the one complete block below.
It obtains the identity and credential itself, so it also works in a newly elevated window. On a fresh
installation it calls the packaged PostgreSQL helper and prompts for the necessary database
passwords before downloading the release. It then creates the protected server configuration
automatically; do not create or edit `server.env` first.

The installer prints the phase and total elapsed time whenever it begins work or is still waiting.
It also records the complete session in
`%ProgramData%\WinnerZone\TermSheet\installation-logs\bootstrap-<timestamp>-<operation>-<attempt>.log`. If installation
stops, the last screen clearly identifies the failed phase, the exact error, the script location,
the transcript path, and the service-log directory. Completed phases are retained, so leave the
window open, correct the reported condition, and rerun the same block. Do not delete an installation
directory to retry. When requesting support, send the newest installation transcript and only the
named `.err.log`; neither file should contain the password entered into the secure credential dialog.

The wrapper maintains `%ProgramData%\WinnerZone\TermSheet\installation\installation-state.json`
as an atomic schema-3 journal. It records the operation, immutable attempt history, exact baseline
release identity, health evidence, consent decision, one-shot request ID, failure policy and
transcripts. Only one Step 3 session can hold the installation lock at a time.

The block asks whether two custodians' distinct public `.cer` files are ready. Answer `Y` to use the
files at the shown paths; their private keys remain with the custodians. Answer `N` to install without
custodians for now. That answer selects the bootstrap's explicit deferred-escrow mode automatically.
The packaged installer retains every answer for one automatic recovery attempt and runs the complete
repair/health check; there is no second command. When an operational failure is eligible for a
software reinstall, it explains exactly what will be replaced and waits for explicit approval.
That approval creates one request which the updater marks consumed before downloading anything. A
failure, restart or scheduled-task rerun cannot reuse it; another attempt requires fresh approval.

```powershell
$servicePolicy = Get-ExecutionPolicy -Scope Process
if ($servicePolicy -ne "Bypass") { Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force }
Get-ChildItem -LiteralPath $PWD -Recurse -File | Unblock-File
$manifestUri = "https://raw.githubusercontent.com/andrelch/term-sheet-extractor-dist/main/production.json"
$global:LASTEXITCODE = 0
& .\install-windows-server.ps1 -ManifestUri $manifestUri
if ($LASTEXITCODE -eq 2) {
  Write-Warning "Step 3 is degraded: the verified prior version is healthy, but the reinstall still needs attention."
  return
}
if ($LASTEXITCODE -ne 0) { throw "Installation did not complete. Use the recovery action printed above." }
```

This is not a recovery mechanism. Until escrow is added, losing the Windows machine or its key
provider can make every encrypted document permanently unreadable. Record who approved the
exception, protect system backups, and appoint custodians as soon as possible.

What each required flag is:

| Flag | What it is |
| --- | --- |
| `-ApplicationRoot` | Where releases are installed. Not where your data lives. |
| `-NssmPath`, `-CaddyPath` | Paths to the tools you installed in [Step 1](#step-1-before-you-start). |
| `-PgBin` | Optional PostgreSQL 18 command-line-tools directory. Defaults to `C:\Program Files\PostgreSQL\18\bin`; use this when IT installed the approved tools elsewhere. Bootstrap records the absolute `pg_dump` and `pg_restore` paths for services and the SYSTEM updater task. |
| `-PublicHostname` | Defaults to `termsheetextractor.local`. Step 3 puts the current LAN address in the employee installer; setup replaces older mappings for that hostname on each employee computer automatically. Pass `localhost` explicitly for a server-only installation. Caddy also keeps `https://localhost/` available for the server-console launcher. |
| `-ClientFilesRoot` | Optional output directory for the verified combined employee installer and its handoff files. Defaults to `C:\TermSheet\Client Files`. |
| `-ServiceUser` | The Windows account the application services run as. The block defaults to the signed-in account; an approved dedicated low-privilege service identity may be substituted. |
| `-EscrowDirectory`, `-EscrowCertificatePath` | Where sealed, encrypted backups of the document-encryption master key are written, and the public certificates of two separate recovery holders. Each certificate produces an independently recoverable package. |
| `-AllowDeferredEscrow` | Explicitly installs without recovery packages when custodians do not yet exist. It cannot be combined with either escrow argument and must be treated as a temporary, approved data-loss risk. |
| `-ManifestUri` | The vendor's update-channel URL. Fixed — it's the same URL for every future update too. |
| `-ReleasePublicKeyPath` | The key authenticated automatically in [Step 2](#step-2-prepare-and-verify-the-bootstrap-package). |

It's safe to run more than once, including from a newly downloaded bootstrap package. Treat a newer
bootstrap as an authoritative maintenance installation: Step 3 does not return successfully until
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
newer release; a queued, failed or still-old result makes Step 3 fail rather than print installation
success. Existing data, secrets, configuration, signing
trust, and rollback releases remain intact. Bootstrap refuses to trust a different signing key on a
rerun — key rotation is a separate, deliberate procedure the vendor will walk you through.

When a higher verified bootstrap finds an updater transaction that is provably still at `staged`,
`quiesced`, or `snapshotting`, it offers a **safe pre-migration takeover**. After operator approval,
the bootstrap replaces the external updater with its bundled, hashed copy, verifies both scheduled
task actions, records provenance, restores the unchanged baseline, and retries every normal snapshot,
migration and health gate. It never takes over a transaction that reached migration or whose baseline
identity changed. This is the supported recovery for the 0.3.15 `Invalid JSON primitive: WARNING`
snapshot-output bug and similar failures left by older updaters.

If bootstrap stops, use the red **INSTALLATION STOPPED** summary rather than searching backward
through the console. For an operational failure, Step 3 offers one automatic software reinstall and
continues only after explicit approval. It redownloads signed application software and recreates
services, tasks, the proxy and launchers while preserving the database, documents, keys,
configuration, backups and logs.

Snapshot, schema/database-state, or key-custody failures may instead offer an
**archive-then-empty reset**. This requires typing the displayed operation-specific confirmation.
The helper creates a redacted support bundle and recovery manifest, renames the existing PostgreSQL
database to a timestamped `term_sheet_extractor_retired_*` name, archives the application, state and
document-store paths, and installs a new empty database and document store. PostgreSQL 18, its Windows
service, and the existing `postgres`, `term_sheet_extractor_admin`, and `term_sheet_app` accounts and
passwords are not removed or changed. Signing-key, release-signature/archive, unsafe-path, BitLocker,
concurrency, and unverifiable-bootstrap failures never offer this reset; a reset cannot make an
untrusted package safe. Do not delete `C:\TermSheet`, the data directory, or
`%ProgramData%\WinnerZone\TermSheet` manually.

An interrupted snapshot uses an operation-bound directory under
`%ProgramData%\WinnerZone\TermSheet\pre-migration-snapshots`. Incomplete data is removed only when
its marker matches the interrupted operation; a completed, verified snapshot is preserved. After
correcting the package or PostgreSQL problem, rerun the exact same Step 3 command. The updater
recovers the pre-migration transaction, restores the previously active release and services, and
starts a fresh operation without applying a migration from the failed attempt.

Before attempting either install, Step 3 records the exact active release directory and marker hash,
its core health, updater/task/proxy/handoff files and service state. A failed reinstall restores that
software baseline before degraded status is considered. Degraded status is allowed only when the
same directory and marker—not merely the same version number—passes structured HTTP and HTTPS,
database migration and both worker checks.
The updater stage is transaction-safe. Bootstrap may retry transient download or preflight failures,
but a pre-migration snapshot failure runs once and is recorded as non-retryable; repeating the same
dump cannot repair invalid evidence, a missing tool, or a watchdog expiration. Each attempt verifies
both enabled scheduled tasks and a readable versioned state file, so a transient ACL or Task
Scheduler failure cannot be mistaken for a completed installation.
After services and launchers are ready, Step 3 starts both tasks under their installed `SYSTEM`
identity. It waits for a successful fresh channel check (and any offered signed update), verifies the
task exit results and updater state, then checks server readiness again. A large first update can keep
Step 3 open while it downloads, backs up, and validates the release; do not close that window. While
the database snapshot runs, the periodic message shows its phase, processed bytes, elapsed time, and
last byte-or-phase activity. The snapshot is terminated before migration after three minutes without
byte progress or 30 minutes total. Bootstrap allows the snapshot process 60 seconds to report and
clean up after either deadline, then safely stops it only if the transaction still proves migration
has not begun.
Immediately before that maintenance update, Step 3 quiesces the extraction and backup workers. It
uses the official Windows service-stop path and waits for NSSM to finish the registered shutdown
sequence without imposing an additional installer timeout or killing worker processes. The workers
are restarted after activation or restored if the installation fails, so background work cannot
outrank installation and is not left disabled.

When the two public certificates become available, add escrow to the existing key from an elevated
PowerShell session. This command does not rotate the key or rewrite encrypted documents:

```powershell
$escrowDirectory = Read-Host "Enter the absolute escrow directory on your connected custody drive/share"
.\add-server-key-escrow.ps1 `
  -EscrowDirectory $escrowDirectory `
  -EscrowCertificatePath C:\KeyBootstrap\custodian-a.cer,C:\KeyBootstrap\custodian-b.cer
```

Give each recovery holder their matching `.p7m` package, verify its SHA-256 value against
`%ProgramData%\WinnerZone\TermSheet\installation-key.json`, and remove the temporary public
certificate copies from the server.

The escrow location is an explicit custody decision: no `E:` drive is assumed and a missing drive
never silently falls back to the server disk. Connect the approved custody drive/share first.
Path and write checks run before the existing document key is decrypted. If a prior attempt left
CMS files but did not finish updating the marker, retain those files and use a new empty destination
or reconcile the prior attempt; do not regenerate the document key.

### If archive reset stops partway through

The recovery manifest path is printed before services stop and persisted in the installation journal.
It records the database retirement stage and each source/destination before and after a rename.
Failures report the exact stage and paths. A missing tool, inaccessible drive, overlapping root or
unsupported cross-volume archive is rejected before database retirement. Directory archives use
same-volume renames without traversing release junctions; archived release-link targets are recorded
in the manifest and the links detached, so they cannot point back into a new live installation.
The installer accepts `-RecoveryParent` for installations whose managed roots share a different
volume; the recovery directory must be separate from every managed root. Mixed-volume managed
roots require administrator-led recovery rather than an automatic copy/delete reset.

If database retirement or file movement has started, a rerun refuses to overwrite the recovery
journal. Keep the entire recovery directory and have the server administrator reconcile the
manifest, archived files and retired database first. Do not delete the journal to force another
reset. This also applies to an interrupted 0.3.17 reset whose outcome cannot be proven. A failed
preflight with explicit evidence that neither database retirement nor file movement began can be
retried after correcting its cause. PostgreSQL accounts, passwords and its service remain unchanged.

## Automatic installation confirmation

Four Windows services, all managed by NSSM, all set to restart automatically:

| Service | What it does |
| --- | --- |
| `TermSheetWeb` | The application itself |
| `TermSheetWorker` | Background processing (document extraction, etc.) |
| `TermSheetProxy` | Caddy — terminates HTTPS on port 443 |
| `TermSheetBackup` | Scheduled database/document backups |

Logs for each are at `%ProgramData%\WinnerZone\TermSheet\logs\<ServiceName>.log` (and `.err.log` for
errors). The Step 3 block already runs the repair helper after installation.
It stops unless every required check passes. Do not copy or run another PowerShell block here;
continue only after Step 3
prints `Installation and automatic server checks passed.`

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

For ordinary version checks after the automatic checks pass, use **Settings → System → Server version and
updates**. If the helper reports a problem, follow its exact recovery action and inspect the named
`.err.log`; do not continue to owner setup until the helper exits successfully.

The bootstrap has already installed `Term Sheet Extractor - Server` in the current and all-users
Desktop and Start menu locations, so it remains visible even when Step 3 ran elevated. Open it now
and confirm the sign-in page loads. This is the supported way to
launch the UI directly from the server computer, including a localhost-only installation.

## Step 4: Finish setup in the interface

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

## Employee installer handoff

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

The Step 3 health check already compares the executable with
`Term-Sheet-Extractor-Employee-Setup.exe.sha256` and refuses to finish if the handoff is incomplete or
does not match. No one needs to compute or enter that hash manually. Keep `server-url.txt` beside the
installer when running setup: the installer copies this installation-specific server address beside
the installed executable automatically. IT can deploy the same file there later when retargeting is
required. Do not copy `server.env`, `ProgramData\WinnerZone\TermSheet\config`, or any server secret to
an employee computer.

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
| `status` | What the updater is doing right now: `idle` (nothing to do), `checking`, `downloading`, `staging`, `snapshotting` (taking a pre-update backup), `activating`, `rolling-back`, `blocked`, or `failed`. |
| `installedVersion` | The version currently serving traffic. |
| `previousVersion` | What was running before the last successful update — the rollback target if needed. |
| `blockedVersion` | A version the unattended updater tried and rejected (failed its health check, or a rollback failed). It will not retry this exact version automatically. Prefer a newer corrected signed release; an administrator may instead run the current Step 3 package and explicitly approve its software reinstall. |
| `activeReleaseUnhealthy` | If `true`, the currently active release failed its own health checks and an automatic rollback wasn't possible (most often because a database migration can't safely run backwards). This is the one status worth paging someone over — the server may be degraded and needs the vendor's attention, ideally restored from the backup noted in `lastSuccessfulUpdateAt`. |
| `lastCheckedAt`, `lastSuccessfulUpdateAt`, `lastFailureAt` | Timestamps, for confirming the updater is actually running on schedule. |
| `progressDetail`, `progressBytes`, `progressUpdatedAt` | Live detail for long-running work. During `snapshotting`, Step 3 reports the dump or verification phase, bytes processed and elapsed time every 15 seconds. These fields clear when the snapshot completes. |
| `retryable` | `false` means repeating the same updater attempt cannot repair the failure. Step 3 stops after that task result instead of repeating the full download and snapshot cycle. Missing on older updater state, or `null`, leaves the existing guarded retry policy in control. |
| `bootstrapMinimumVersion`, `updaterProvenanceSha256` | Identify the verified maintenance bootstrap that installed the external updater and the hash of its provenance manifest. A missing or changed manifest on a current installation stops the updater and requires rerunning the verified Step 3 package. |

Pre-migration snapshots have two independent watchdogs. The managed defaults stop `pg_dump` after
three minutes without byte progress and stop the complete snapshot after 30 minutes. Either failure
happens before `prisma migrate deploy`, removes the incomplete snapshot directory, restores services
when schema-aware recovery is not required, and records `TSE-STEP3-SNAPSHOT` with `retryable: false`.
This recovery point is never bypassed. Correct the reported storage, PostgreSQL or sizing problem and
rerun the same Step 3 command; completed installation phases and the previously active release remain
preserved.

`blocked` is not a stale download cache: the channel manifest is fetched with cache revalidation and
verified on every check. The updater retains the original `failureReason`, `lastFailureAt` and
`rollbackPerformed` evidence while holding the exact failed version, writes the reason to stderr and
returns a nonzero scheduled-task result. Do not delete the state file. Once the vendor publishes a
corrected, higher signed version, the existing scheduled updater evaluates it as a new candidate and
installs it through the normal backup and health gates. If the operator instead approves Step 3's
software reinstall, the same signed version is redownloaded and passes those gates again. A
successful activation clears the obsolete block automatically.

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

If PostgreSQL 18 is missing, reinstall PostgreSQL 18 and rerun Step 3. If a Term Sheet
service, updater task, or launcher file is missing, rerun Step 3 with the same values. Bootstrap
preserves the pinned signing key, installed release, data, and existing authentication secret.

### Stopping the server for NSSM or tool maintenance

An idle `TermSheetBackup` does not imply a backup is running. Older installations gave its
PowerShell wrapper a 30-minute NSSM console-stop timeout, which can leave the service in
`STOP_PENDING` while it prints output. Current releases handle NSSM's Windows `CTRL+BREAK` signal
directly so idle processes exit promptly, and current Step 3 also reduces the forced-stop fallback
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
stopped. After maintenance, rerun Step 3 (preferred, because it reapplies current service settings),
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

From elevated PowerShell in the retained extracted bootstrap package directory, create a redacted
support ZIP with the repair context included:

```powershell
if (-not (Test-Path -LiteralPath .\repair-windows-server.ps1 -PathType Leaf)) {
  throw "Open PowerShell in the retained extracted bootstrap package directory, then rerun this block."
}
.\repair-windows-server.ps1 -AcceptSafeRepairs -CreateSupportBundle
```

To export only the bundle without running checks or repairs, use
`.\export-installation-support-bundle.ps1` from that same directory. Neither bundle includes
documents, configuration secrets, tokens, or database contents.

Contact your vendor with the version from `updater-state.json`'s `installedVersion`, the relevant
lines from the service's `.err.log`, and — if it's update-related — the full `updater-state.json`.
Don't send `server.env` or anything from the `config\` or `secrets\` folders; the vendor should never
need those, and they contain your credentials.
