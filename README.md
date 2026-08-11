# Term Sheet server — operator guide

> Verified against the current bootstrap, updater, release layout, and service installer on
> 11 August 2026.

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
  throw "Close this window and open Windows PowerShell with Run as administrator."
}
```

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

Then configure the service, create separate administrator and application logins, and restrict the
database to this machine. `term_sheet_extractor_admin` owns the database; `term_sheet_app` is only
the runtime login placed in `DATABASE_URL`. The commands prompt for passwords instead of putting
them in shell history. Passwords have no character-set or length policy here; any non-empty value
is accepted. The snippets safely quote passwords for PostgreSQL and percent-encode the application
password when it is placed in `DATABASE_URL` in Step 6. If either role already exists, enter its
current password: setup keeps the role unchanged and proves the password with a real login before
continuing.

```powershell
$pgBin = "C:\Program Files\PostgreSQL\18\bin"
$psql = Join-Path $pgBin "psql.exe"
$pgIsReady = Join-Path $pgBin "pg_isready.exe"
if (-not (Test-Path -LiteralPath $psql)) { throw "PostgreSQL 18 command-line tools were not installed at $pgBin." }

$postgresService = Get-Service -Name "postgresql*18*" | Select-Object -First 1
if (-not $postgresService) { throw "The PostgreSQL 18 Windows service was not found." }
Set-Service -Name $postgresService.Name -StartupType Automatic
if ($postgresService.Status -ne "Running") { Start-Service -Name $postgresService.Name }

$postgresPassword = Read-Host "Enter the postgres administrator password" -AsSecureString
$adminPasswordSecure = Read-Host "Enter the current term_sheet_extractor_admin password, or create one if the role does not exist" -AsSecureString
$appPasswordSecure = Read-Host "Enter the current term_sheet_app password, or create one if the role does not exist" -AsSecureString
$postgresPasswordPlain = [Net.NetworkCredential]::new("", $postgresPassword).Password
$adminDbPassword = [Net.NetworkCredential]::new("", $adminPasswordSecure).Password
$appDbPassword = [Net.NetworkCredential]::new("", $appPasswordSecure).Password
if ($adminDbPassword.Length -eq 0) { throw "The database administrator password cannot be empty." }
if ($appDbPassword.Length -eq 0) { throw "The database password cannot be empty." }

$env:PGPASSWORD = $postgresPasswordPlain
try {
  $adminRoleProbe = & $psql -X -q -t -A -v ON_ERROR_STOP=1 -U postgres -d postgres `
    -c "SELECT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'term_sheet_extractor_admin');"
  if ($LASTEXITCODE -ne 0) { throw "Could not check whether term_sheet_extractor_admin already exists." }
  $adminRoleProbe = ([string]$adminRoleProbe).Trim()
  if ($adminRoleProbe -notin @("t", "f")) { throw "PostgreSQL returned an unexpected administrator-role check result: '$adminRoleProbe'." }
  $adminRoleExists = $adminRoleProbe -eq "t"

  if (-not $adminRoleExists) {
    $adminPasswordSql = $adminDbPassword.Replace("'", "''")
    "CREATE ROLE term_sheet_extractor_admin LOGIN PASSWORD '$adminPasswordSql';" |
      & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres
    $adminPasswordSql = $null
    if ($LASTEXITCODE -ne 0) { throw "Could not create the term_sheet_extractor_admin role." }
  }

  $appRoleProbe = & $psql -X -q -t -A -v ON_ERROR_STOP=1 -U postgres -d postgres `
    -c "SELECT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'term_sheet_app');"
  if ($LASTEXITCODE -ne 0) { throw "Could not check whether term_sheet_app already exists." }
  $appRoleProbe = ([string]$appRoleProbe).Trim()
  if ($appRoleProbe -notin @("t", "f")) { throw "PostgreSQL returned an unexpected application-role check result: '$appRoleProbe'." }
  $appRoleExists = $appRoleProbe -eq "t"

  if (-not $appRoleExists) {
    $appPasswordSql = $appDbPassword.Replace("'", "''")
    "CREATE ROLE term_sheet_app LOGIN PASSWORD '$appPasswordSql';" |
      & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres
    $appPasswordSql = $null
    if ($LASTEXITCODE -ne 0) { throw "Could not create the term_sheet_app role." }
  }

  $env:PGPASSWORD = $adminDbPassword
  & $psql -X -q -v ON_ERROR_STOP=1 -h 127.0.0.1 -U term_sheet_extractor_admin -d postgres `
    -c "SELECT current_user;" | Out-Null
  if ($LASTEXITCODE -ne 0) {
    throw "The entered term_sheet_extractor_admin password was rejected. If the role already existed, enter its current password or reset it as the postgres administrator."
  }

  $env:PGPASSWORD = $appDbPassword
  & $psql -X -q -v ON_ERROR_STOP=1 -h 127.0.0.1 -U term_sheet_app -d postgres `
    -c "SELECT current_user;" | Out-Null
  if ($LASTEXITCODE -ne 0) {
    throw "The entered term_sheet_app password was rejected. If the role already existed, enter its current password or reset it as the postgres administrator."
  }

  $env:PGPASSWORD = $postgresPasswordPlain

  $databaseOwner = & $psql -X -q -t -A -v ON_ERROR_STOP=1 -U postgres -d postgres `
    -c "SELECT COALESCE((SELECT pg_get_userbyid(datdba) FROM pg_database WHERE datname = 'term_sheet_extractor'), '');"
  if ($LASTEXITCODE -ne 0) { throw "Could not inspect the term_sheet_extractor database." }
  $databaseOwner = ([string]$databaseOwner).Trim()
  if (-not $databaseOwner) {
    & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres `
      -c "CREATE DATABASE term_sheet_extractor OWNER term_sheet_extractor_admin;"
    if ($LASTEXITCODE -ne 0) { throw "Could not create the term_sheet_extractor database." }
  } elseif ($databaseOwner -ne "term_sheet_extractor_admin") {
    if ($databaseOwner -eq "postgres") {
      $currentOwnerPasswordPlain = $postgresPasswordPlain
    } elseif ($databaseOwner -eq "term_sheet_app") {
      $currentOwnerPasswordPlain = $appDbPassword
    } else {
      $currentOwnerPasswordSecure = Read-Host "The existing database is owned by '$databaseOwner'. Enter that database owner's password to authorize transfer to term_sheet_extractor_admin" -AsSecureString
      $currentOwnerPasswordPlain = [Net.NetworkCredential]::new("", $currentOwnerPasswordSecure).Password
    }
    if ($currentOwnerPasswordPlain.Length -eq 0) { throw "The current database owner's password cannot be empty; ownership was not changed." }

    $env:PGPASSWORD = $currentOwnerPasswordPlain
    & $psql -X -q -v ON_ERROR_STOP=1 -h 127.0.0.1 -U $databaseOwner -d term_sheet_extractor `
      -c "SELECT current_user;" | Out-Null
    if ($LASTEXITCODE -ne 0) {
      throw "The password for current database owner '$databaseOwner' was rejected; ownership was not changed."
    }

    $env:PGPASSWORD = $postgresPasswordPlain
    & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres `
      -c "ALTER DATABASE term_sheet_extractor OWNER TO term_sheet_extractor_admin;"
    if ($LASTEXITCODE -ne 0) { throw "Could not transfer database ownership to term_sheet_extractor_admin." }
    $currentOwnerPasswordPlain = $null
  }

  & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres `
    -c "GRANT CONNECT, TEMPORARY ON DATABASE term_sheet_extractor TO term_sheet_app;"
  if ($LASTEXITCODE -ne 0) { throw "Could not grant term_sheet_app access to the database." }
  & $psql -X -v ON_ERROR_STOP=1 -U postgres -d term_sheet_extractor `
    -c "GRANT USAGE, CREATE ON SCHEMA public TO term_sheet_app;"
  if ($LASTEXITCODE -ne 0) { throw "Could not grant term_sheet_app access to the public schema." }

  & $psql -X -v ON_ERROR_STOP=1 -U postgres -d postgres `
    -c "ALTER SYSTEM SET listen_addresses = 'localhost';"
  if ($LASTEXITCODE -ne 0) { throw "Could not restrict PostgreSQL to localhost." }
} finally {
  Remove-Item Env:PGPASSWORD -ErrorAction SilentlyContinue
  $postgresPasswordPlain = $null
  $currentOwnerPasswordPlain = $null
  $adminPasswordSql = $null
  $appPasswordSql = $null
}

Restart-Service -Name $postgresService.Name
& $pgIsReady -h 127.0.0.1 -p 5432

$env:PGPASSWORD = $appDbPassword
try {
  & $psql -v ON_ERROR_STOP=1 -h 127.0.0.1 -U term_sheet_app -d term_sheet_extractor `
    -c "SELECT current_user, current_database();"
  if ($LASTEXITCODE -ne 0) {
    throw "The entered term_sheet_app password was rejected. If the role already existed, enter its current password or reset it as a PostgreSQL administrator."
  }
} finally {
  Remove-Item Env:PGPASSWORD -ErrorAction SilentlyContinue
}

if (-not (Get-NetFirewallRule -DisplayName "Block Term Sheet PostgreSQL from LAN" -ErrorAction SilentlyContinue)) {
  New-NetFirewallRule -DisplayName "Block Term Sheet PostgreSQL from LAN" `
    -Direction Inbound -LocalPort 5432 -Protocol TCP -Action Block
}
```

Keep this PowerShell window open until Step 6 so `$appDbPassword` can be written into the server
configuration. Keep the `term_sheet_extractor_admin` password in the organization's password vault;
the running application does not use it. For an existing role, the successful login test is what
proves the entered password is current; PostgreSQL never exposes the stored password. If an older or
manually created database has a different owner, setup asks for that owner's password and verifies
it before transferring only the database ownership. Do not send or screenshot any database password.

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
$serviceCredential = Get-Credential -UserName $serviceUser `
  -Message "Enter the current Windows account password used to run the Term Sheet services"

# This must resolve before bootstrap; otherwise Windows cannot apply its folder permissions.
$serviceAccount = New-Object Security.Principal.NTAccount($serviceUser)
$serviceAccount.Translate([Security.Principal.SecurityIdentifier]).Value
```

If the signed-in account is a domain account, set `$serviceUser` to its real `DOMAIN\username`
instead. A dedicated service account or gMSA remains supported when company policy requires one;
set `$serviceUser` and `$serviceCredential` to that approved identity. The bootstrap resolves the
account to a SID before changing files or services, so a wrong account name fails before installation.

## Step 2: Download and extract the bootstrap package

The distribution repository root intentionally contains only the update channel, signing key, and
this guide. The PowerShell scripts are in the versioned bootstrap ZIP attached to the current server
release. If `preflight-connectivity.ps1` is already beside this guide, you are reading the extracted
copy and can continue to Step 3. Otherwise, open PowerShell in the directory where you want to keep
the installer and run:

```powershell
$bootstrapUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.8.2/term-sheet-bootstrap-0.2.8.2.zip"
$bootstrapChecksumUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.8.2/term-sheet-bootstrap-0.2.8.2.zip.sha256"
$downloadRoot = Join-Path $PWD "term-sheet-bootstrap-0.2.8.2-download"
$bootstrapZip = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.8.2.zip"
$bootstrapChecksum = "$bootstrapZip.sha256"

New-Item -ItemType Directory -Path $downloadRoot -ErrorAction Stop | Out-Null
Invoke-WebRequest -Uri $bootstrapUrl -OutFile $bootstrapZip -UseBasicParsing
Invoke-WebRequest -Uri $bootstrapChecksumUrl -OutFile $bootstrapChecksum -UseBasicParsing

$expectedBootstrapHash = ((Get-Content -LiteralPath $bootstrapChecksum -Raw).Trim() -split '\s+')[0].ToLowerInvariant()
$actualBootstrapHash = (Get-FileHash -LiteralPath $bootstrapZip -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actualBootstrapHash -ne $expectedBootstrapHash) { throw "Bootstrap ZIP checksum mismatch." }

$packageDirectory = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.8.2"
Expand-Archive -LiteralPath $bootstrapZip -DestinationPath $packageDirectory
Set-Location $packageDirectory
Get-ChildItem .\preflight-connectivity.ps1, .\verify-release.ps1, .\bootstrap-windows-server.ps1
Get-ChildItem .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe, `
  .\employee-launcher\Term-Sheet-Extractor-Employee-Setup-*.exe.sha256
```

The commands must list all three scripts plus one employee installer and its checksum. Run every
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

`preflight-connectivity.ps1` confirms the machine is ready before you commit to an install. It makes
no changes, needs no administrator rights, and reports every problem it finds in one pass rather than
stopping at the first one.

The distribution copy of this guide fills in the exact manifest URL and signing-key fingerprint
when a release is published. Keep both values quoted if you copy this template from a source
checkout: PowerShell treats an unquoted `<placeholder>` as an operator.

```powershell
$manifestUri = "https://raw.githubusercontent.com/andrelch/term-sheet-extractor-dist/main/production.json"
$expectedFingerprint = "54ce5bf97695f05fa2223e6e8320d4b91445513e7210028863136e8faa833217"

.\preflight-connectivity.ps1 -ManifestUri $manifestUri `
  -PublicKeyPath .\release-signing-public.pem -ExpectedPublicKeyFingerprint $expectedFingerprint `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe
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

## Step 6: Create the server configuration

The bootstrap requires the persistent configuration file to exist before it runs. Create it from
the complete example included in the extracted package, insert the database password created in
Step 1, and generate a separate authentication secret:

```powershell
$configFile = Join-Path $env:ProgramData "WinnerZone\TermSheet\config\server.env"
New-Item -ItemType Directory -Force -Path (Split-Path -Parent $configFile) | Out-Null
Copy-Item -LiteralPath .\server.env.example -Destination $configFile -ErrorAction Stop

if ([string]::IsNullOrEmpty($appDbPassword)) {
  $appPasswordSecure = Read-Host "Re-enter the term_sheet_app database password from Step 1" -AsSecureString
  $appDbPassword = [Net.NetworkCredential]::new("", $appPasswordSecure).Password
}
if ($appDbPassword.Length -eq 0) { throw "The database password cannot be empty." }

$authBytes = New-Object byte[] 48
$random = [Security.Cryptography.RandomNumberGenerator]::Create()
try { $random.GetBytes($authBytes) } finally { $random.Dispose() }
$authSecret = [Convert]::ToBase64String($authBytes)
$encodedAppDbPassword = [Uri]::EscapeDataString($appDbPassword)
$databaseUrl = "postgresql://term_sheet_app:$encodedAppDbPassword@127.0.0.1:5432/term_sheet_extractor?schema=public"

$config = Get-Content -LiteralPath $configFile -Raw
$config = [regex]::Replace($config, '(?m)^DATABASE_URL=.*$', ('DATABASE_URL="' + $databaseUrl + '"'))
$config = [regex]::Replace($config, '(?m)^AUTH_SECRET=.*$', ('AUTH_SECRET=' + $authSecret))
$config = [regex]::Replace($config, '(?m)^DOCUMENT_STORE_ROOT=.*$', 'DOCUMENT_STORE_ROOT=C:\TermSheetData\objects')
[IO.File]::WriteAllText($configFile, $config, (New-Object Text.UTF8Encoding($false)))

$appDbPassword = $null
$encodedAppDbPassword = $null
$authSecret = $null
notepad.exe $configFile
```

In Notepad, fill in the organization's authentication settings and any approved provider settings.
Never add `DOCUMENT_STORE_MASTER_KEY`; the bootstrap rejects it because the central encryption key
must be held by DPAPI or Azure Key Vault. Save the file, close Notepad, and confirm the required
values are no longer blank. Do not print the file to the console:

```powershell
$requiredNames = @("DATABASE_URL", "AUTH_SECRET")
$missingNames = foreach ($name in $requiredNames) {
  if (-not (Select-String -LiteralPath $configFile -Pattern "^$name=.+" -Quiet)) { $name }
}
if ($missingNames) { throw "Missing server.env values: $($missingNames -join ', ')" }
if (Select-String -LiteralPath $configFile -Pattern '^[ \t]*DOCUMENT_STORE_MASTER_KEY[ \t]*=[ \t]*\S+' -Quiet) {
  throw "Remove DOCUMENT_STORE_MASTER_KEY from server.env."
}
```

Choose how provider API keys will be owned:

- **Application-managed (the default):** leave `SECRET_STORE=encrypted` and leave the five provider
  variables empty. After installation, a trusted application administrator with `data-admin` enters
  approved keys under **Settings -> API keys**.
- **IT-managed:** set `SECRET_STORE=environment` and place only approved credentials in
  `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `DEEPSEEK_API_KEY`, or `NVIDIA_API_KEY`.
  Restart the web and worker services after an IT-managed key changes.

Telegram and browser notification credentials follow the same ownership choice. Do not send
`server.env` to the vendor or include it in a ticket or screenshot.

## Step 7: Install

From an elevated PowerShell session, in this package's directory:

```powershell
$publicHostname = Read-Host "Enter the employee DNS hostname, or press Enter to use localhost on this server only"
if (-not $publicHostname) { $publicHostname = "localhost" }
if ($publicHostname -notmatch '^[A-Za-z0-9](?:[A-Za-z0-9.-]*[A-Za-z0-9])?$' -or $publicHostname.Contains('..')) {
  throw "Enter a valid DNS hostname or localhost."
}

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
errors). Confirm all four services are running and the updater wrote its initial state:

```powershell
$serviceNames = @("TermSheetWeb", "TermSheetWorker", "TermSheetProxy", "TermSheetBackup")
$services = Get-Service -Name $serviceNames
$services | Format-Table Name, Status, StartType
if ($services.Where({ $_.Status -ne "Running" })) { throw "One or more Term Sheet services are not running." }

$stateFile = Join-Path $env:ProgramData "WinnerZone\TermSheet\updater-state.json"
if (-not (Test-Path -LiteralPath $stateFile)) { throw "Updater state was not created at $stateFile." }
Get-Content -LiteralPath $stateFile -Raw | ConvertFrom-Json

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
`Term-Sheet-Extractor-Employee-Setup.exe.sha256`. `server-url.txt` records the installation-specific
server address; the launcher's distribution mechanism can deploy it beside the installed executable
when retargeting is required. Do not copy `server.env`, `ProgramData\WinnerZone\TermSheet\config`, or
any server secret to an employee computer.

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

To check status at any time:

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

To start or verify everything manually, open **Windows PowerShell as Administrator** and run:

```powershell
$postgresService = Get-Service -Name "postgresql*18*" | Select-Object -First 1
if (-not $postgresService) { throw "The PostgreSQL 18 service was not found." }
if ($postgresService.Status -ne "Running") { Start-Service -Name $postgresService.Name }

$serviceNames = @("TermSheetWeb", "TermSheetWorker", "TermSheetBackup", "TermSheetProxy")
foreach ($name in $serviceNames) {
  $service = Get-Service -Name $name -ErrorAction Stop
  if ($service.Status -ne "Running") { Start-Service -Name $name }
}
Get-Service -Name (@($postgresService.Name) + $serviceNames) | Format-Table Name, Status, StartType

$localLauncher = "C:\TermSheet\Client Files\Term Sheet Extractor - Server.url"
if (-not (Test-Path -LiteralPath $localLauncher)) {
  throw "The server UI launcher is missing at $localLauncher. Rerun bootstrap to regenerate it."
}
Start-Process $localLauncher
```

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
`server-url.txt`. Confirm the generated file contains the correct HTTPS URL, then have the launcher
distribution mechanism place that file beside the installed executable:

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
