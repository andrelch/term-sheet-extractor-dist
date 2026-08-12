# Term Sheet server — operator guide

> Verified against the current bootstrap, updater, release layout, and service installer on
> 12 August 2026.

This is for the person installing and running the Term Sheet server on your own Windows machine.
It covers installation, day-to-day operation, and what to do when something goes wrong. It does not
assume access to the vendor's source code or internal engineering documentation — you shouldn't need
either.

If anything here doesn't match what you're seeing, stop and contact the vendor rather than guessing.

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

If elevation is declined, nothing is changed. Accept it later and restart Step 1; every installation
and check below is safe to rerun.

### Step 1.1: Install Node.js 22 x64

Node.js is needed only to verify and install the first release. Installed releases use their own
bundled, tested `runtime\node.exe`. Do not install npm packages, Git, or the source repository on the
server.

On Windows Server 2025 with WinGet available, install the version-specific Node 22 package. Do not
use `OpenJS.NodeJS.LTS`, because that package moves to a new major when the active LTS line changes.

```powershell
winget source update
winget install --id OpenJS.NodeJS.22 --exact --source winget --architecture x64 --scope machine `
  --accept-source-agreements --accept-package-agreements

# Refresh PATH in this PowerShell window, then verify both version and architecture.
$env:Path = [Environment]::GetEnvironmentVariable("Path", "Machine") + ";" + `
  [Environment]::GetEnvironmentVariable("Path", "User")
node.exe --version
node.exe -p "process.arch"
```

The version must start with `v22.` and the architecture must be `x64`. If `winget.exe` is not
available (for example, on Windows Server 2022), download the **Windows x64 MSI for Node.js 22** from
the [official Node.js download page](https://nodejs.org/en/download), run the MSI as administrator,
accept the default machine-wide PATH option, open a new elevated PowerShell window, and run the two
verification commands above.

If WinGet says Node.js 22 is already installed, do not uninstall it: refresh `PATH` and run the two
checks again. If either check is still wrong, repair the existing Node.js 22 x64 installation from
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
paths in every later command:

```powershell
Get-Item C:\tools\nssm.exe, C:\tools\caddy.exe
& C:\tools\nssm.exe version
& C:\tools\caddy.exe version
```

By default, use the currently signed-in Windows account and retain its credential in this PowerShell
session. `COMPUTERNAME\USERNAME` is intentional and avoids asking the client to create another local
account solely for setup:

```powershell
$serviceUser = "$env:COMPUTERNAME\$env:USERNAME"
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
$bootstrapUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.9.6/term-sheet-bootstrap-0.2.9.6.zip"
$bootstrapChecksumUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.9.6/term-sheet-bootstrap-0.2.9.6.zip.sha256"
$downloadRoot = Join-Path $PWD "term-sheet-bootstrap-0.2.9.6-download"
$bootstrapZip = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.9.6.zip"
$bootstrapChecksum = "$bootstrapZip.sha256"

$packageDirectory = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.9.6"
New-Item -ItemType Directory -Path $downloadRoot -Force | Out-Null
if (-not (Test-Path -LiteralPath (Join-Path $packageDirectory "preflight-connectivity.ps1"))) {
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
      if (Test-Path -LiteralPath $packageDirectory) { $packageDirectory = "$packageDirectory-$(Get-Date -Format yyyyMMddHHmmss)" }
      Move-Item -LiteralPath $stagingDirectory -Destination $packageDirectory
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
Get-ChildItem .\preflight-connectivity.ps1, .\verify-release.ps1, `
  .\configure-postgresql18.ps1, .\bootstrap-windows-server.ps1
Get-ChildItem .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe, `
  .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe.sha256
```

The commands must list all four scripts plus one employee installer and its checksum. Run every
remaining command from that extracted package directory.

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

Fix everything reported before continuing — each one will surface again, less clearly, during actual
install or first update.

The JSON result includes a `remediation` for every failed check. With `-OfferRemediation`, the script
also prints those actions and can install the pinned Node.js 22 x64 package after confirmation when
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
the selected DPAPI or Key Vault provider, and local installation-owner access. It never prints a
password or writes a temporary password file.

The helper safely handles fresh installs, reruns, existing roles, and legacy or third-party database
owners. Empty, mismatched, or rejected passwords prompt again with a specific recovery action.
For a PostgreSQL owner created with `NOLOGIN`, the authenticated `postgres` administrator must type `TRANSFER` explicitly before ownership changes.
The bootstrap preserves an existing configuration and authentication secret on every rerun. It also
rejects a populated `DOCUMENT_STORE_MASTER_KEY`, because the central key belongs in DPAPI or Azure
Key Vault rather than in an environment file.

Provider API keys deliberately remain blank. After installation, use the localhost server launcher,
choose **Developer access**, and enter approved keys under **Settings -> API keys**. The default
`SECRET_STORE=encrypted` stores them through the application's encrypted secret store. Staff and
provider configuration therefore require no Notepad session and no direct access to `server.env`.

## Step 7: Install

From an elevated PowerShell session, in this package's directory, run the command below. On a fresh
installation it calls the packaged PostgreSQL helper and prompts for the necessary database
passwords before downloading the release. It then creates the protected server configuration
automatically; do not create or edit `server.env` first.

```powershell
do {
  $publicHostname = Read-Host "Enter the employee DNS hostname, or press Enter to use localhost on this server only"
  if (-not $publicHostname) { $publicHostname = "localhost" }
  $hostnameIsValid = $publicHostname -match '^[A-Za-z0-9](?:[A-Za-z0-9.-]*[A-Za-z0-9])?$' -and -not $publicHostname.Contains('..')
  if (-not $hostnameIsValid) { Write-Warning "That is not a valid DNS hostname. Try again or press Enter for localhost." }
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
| `-PublicHostname` | Defaults to `localhost`, which supports using the UI on the server computer. Supply a DNS hostname when employee computers must connect over the network. Caddy also keeps `https://localhost/` available for the server-console launcher. |
| `-ClientFilesRoot` | Optional output directory for the verified combined employee installer and its handoff files. Defaults to `C:\TermSheet\Client Files`. |
| `-ServiceUser` | The low-privilege Windows account the application services run as. Not an administrator account — the bootstrap deliberately restricts what this account can touch. |
| `-EscrowDirectory`, `-EscrowCertificatePath` | Where sealed, encrypted backups of the document-encryption master key are written, and the public certificates of two separate recovery holders. Each certificate produces an independently recoverable package. |
| `-AllowDeferredEscrow` | Explicitly installs without recovery packages when custodians do not yet exist. It cannot be combined with either escrow argument and must be treated as a temporary, approved data-loss risk. |
| `-ManifestUri` | The vendor's update-channel URL. Fixed — it's the same URL for every future update too. |
| `-ReleasePublicKeyPath` | The key from [Step 3](#step-3-confirm-you-have-the-right-key). |

It's safe to run more than once. If a release is already installed, it leaves it alone and reports
that. It refuses to trust a different signing key on a rerun — if you need to install a new key,
that's a separate, deliberate procedure the vendor will walk you through, not something this script
does implicitly.

If bootstrap stops, keep its error message, apply the specific fix it proposes, and rerun the same
Step 7 command. It resumes from verified state and recreates missing services or launcher files; do
not delete `C:\TermSheet`, the data directory, or `%ProgramData%\WinnerZone\TermSheet` to retry.

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
errors). Confirm all four services are running, the updater wrote its initial state, and the
launcher handoff exists. The repair helper reports all problems, offers safe starts, and proposes
an idempotent bootstrap rerun for missing components. For ordinary version checks, use
**Settings → System → Server version and updates**; the commands below are the recovery fallback:

```powershell
.\repair-windows-server.ps1

$stateFile = Join-Path $env:ProgramData "WinnerZone\TermSheet\updater-state.json"
if (Test-Path -LiteralPath $stateFile) { Get-Content -LiteralPath $stateFile -Raw | ConvertFrom-Json }

Get-NetTCPConnection -State Listen | Where-Object LocalPort -In 443, 3000, 5432 |
  Format-Table LocalAddress, LocalPort, OwningProcess

$clientFilesRoot = "C:\TermSheet\Client Files"
Get-ChildItem -LiteralPath $clientFilesRoot
Get-Content -LiteralPath (Join-Path $clientFilesRoot "server-url.txt") -Raw
Get-FileHash -Algorithm SHA256 `
  -LiteralPath (Join-Path $clientFilesRoot "Term-Sheet-Extractor-Employee-Setup.exe")
```

Port `443` is the HTTPS entry point. Ports `3000` and `5432` must listen only on loopback, never on a
LAN address. The client-files listing must include `Term-Sheet-Extractor-Employee-Setup.exe`, its
`.sha256` file, `server-url.txt`, `Term Sheet Extractor - Server.url`, `README.txt`, and
`Term-Sheet-Client-Files.zip`. The hash must match the `.sha256` file and `server-url.txt` must match
the `$publicHostname` supplied in Step 7. If anything looks wrong, inspect the corresponding
`.err.log` first.

The bootstrap has already installed `Term Sheet Extractor - Server` on the current Windows user's
Desktop and Start menu. Open it now and confirm the sign-in page loads. This is the supported way to
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
