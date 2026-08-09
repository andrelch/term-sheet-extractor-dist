# Term Sheet server — operator guide

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

## Before you start

Install these first, from packages you trust:

- PostgreSQL 18 (x64)
- Node.js 22 LTS (x64) — the server release ships without its own Node runtime
- [NSSM](https://nssm.cc/) — runs the services
- [Caddy](https://caddyserver.com/) — terminates HTTPS

You'll also need: an elevated PowerShell session, a DNS hostname pointed at this machine, and two
people available to hold escrow certificates (see [Step 4](#step-4-install)).

## Step 0: Download and extract the bootstrap package

The repository root intentionally contains only the update channel, signing key, and this guide.
The PowerShell scripts are in the versioned bootstrap ZIP attached to the current server release.
Open PowerShell in the directory where you want to keep the installer and run:

```powershell
$bootstrapUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.5/term-sheet-bootstrap-0.2.5.zip"
$bootstrapChecksumUrl = "https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-v0.2.5/term-sheet-bootstrap-0.2.5.zip.sha256"
$downloadRoot = Join-Path $PWD "term-sheet-bootstrap-0.2.5-download"
$bootstrapZip = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.5.zip"
$bootstrapChecksum = "$bootstrapZip.sha256"

New-Item -ItemType Directory -Path $downloadRoot -ErrorAction Stop | Out-Null
Invoke-WebRequest -Uri $bootstrapUrl -OutFile $bootstrapZip -UseBasicParsing
Invoke-WebRequest -Uri $bootstrapChecksumUrl -OutFile $bootstrapChecksum -UseBasicParsing

$expectedBootstrapHash = ((Get-Content -LiteralPath $bootstrapChecksum -Raw).Trim() -split '\s+')[0].ToLowerInvariant()
$actualBootstrapHash = (Get-FileHash -LiteralPath $bootstrapZip -Algorithm SHA256).Hash.ToLowerInvariant()
if ($actualBootstrapHash -ne $expectedBootstrapHash) { throw "Bootstrap ZIP checksum mismatch." }

$packageDirectory = Join-Path $downloadRoot "term-sheet-bootstrap-0.2.5"
Expand-Archive -LiteralPath $bootstrapZip -DestinationPath $packageDirectory
Set-Location $packageDirectory
Get-ChildItem .\preflight-connectivity.ps1, .\verify-release.ps1, .\bootstrap-windows-server.ps1
```

The last command must list all three scripts. Run every remaining command from that extracted
package directory.

## Step 1: Confirm you have the right key

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

## Step 2: Check the machine

`preflight-connectivity.ps1` confirms the machine is ready before you commit to an install. It makes
no changes, needs no administrator rights, and reports every problem it finds in one pass rather than
stopping at the first one.

The release manifest URL and current signing-key fingerprint are filled in below. No placeholders
need to be replaced; keep the quoted values intact when pasting into PowerShell. Confirm the
fingerprint still matches the value supplied through the separate channel described in Step 1.

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

## Step 3: Verify the release before installing

`verify-release.ps1` downloads the exact release your key would install, checks its signature,
size, and checksum, and confirms its release marker — the same checks the installer and every future
update apply — then deletes the download. It changes nothing on the machine.

```powershell
.\verify-release.ps1 -ManifestUri $manifestUri -PublicKeyPath .\release-signing-public.pem
```

A clean run prints the version and release identifier it verified. Any failure here means don't
proceed with install — something about the release doesn't check out, and continuing anyway would
just fail the same check again during installation, or worse, not fail it.

## Step 4: Install

From an elevated PowerShell session, in this package's directory:

```powershell
.\bootstrap-windows-server.ps1 -ApplicationRoot C:\TermSheet `
  -NssmPath C:\tools\nssm.exe -CaddyPath C:\tools\caddy.exe `
  -PublicHostname termsheets.example.com -ServiceUser 'CORP\svc-termsheet' `
  -EscrowDirectory D:\Escrow -EscrowCertificatePath D:\Escrow\custodian-a.cer,D:\Escrow\custodian-b.cer `
  -ManifestUri $manifestUri -ReleasePublicKeyPath .\release-signing-public.pem
```

What each required flag is:

| Flag | What it is |
| --- | --- |
| `-ApplicationRoot` | Where releases are installed. Not where your data lives. |
| `-NssmPath`, `-CaddyPath` | Paths to the tools you installed in [Before you start](#before-you-start). |
| `-PublicHostname` | The DNS name customers/staff will reach this server at. Caddy provisions HTTPS for it automatically. |
| `-ServiceUser` | The low-privilege Windows account the application services run as. Not an administrator account — the bootstrap deliberately restricts what this account can touch. |
| `-EscrowDirectory`, `-EscrowCertificatePath` | Where a sealed, encrypted backup of the document-encryption master key is written, and the public certificates of two separate people who can each decrypt it. This key is generated once and never leaves this machine except as this sealed escrow copy — losing it without an escrow copy means losing access to every stored document. Two certificates are required on purpose: no single person can recover it alone. |
| `-ManifestUri` | The vendor's update-channel URL. Fixed — it's the same URL for every future update too. |
| `-ReleasePublicKeyPath` | The key from [Step 1](#step-1-confirm-you-have-the-right-key). |

It's safe to run more than once. If a release is already installed, it leaves it alone and reports
that. It refuses to trust a different signing key on a rerun — if you need to install a new key,
that's a separate, deliberate procedure the vendor will walk you through, not something this script
does implicitly.

Before running it, create `%ProgramData%\WinnerZone\TermSheet\config\server.env` from
`server.env.example` (also in this package). Leave `DOCUMENT_STORE_MASTER_KEY` out of it entirely —
the bootstrap refuses to continue if it's present; the key is generated and held by the machine
itself (via DPAPI, or Azure Key Vault if you passed `-KeyProvider AzureKeyVault`).

Choose how provider API keys will be owned:

- **Application-managed (the default):** leave `SECRET_STORE=encrypted` and leave the five provider
  variables empty. After installation, a trusted application administrator with `data-admin` enters
  approved keys under **Settings -> API keys**. They are encrypted in PostgreSQL under the same
  machine-held master key; a Settings key overrides a matching environment value.
- **IT-managed:** set `SECRET_STORE=environment` and place only approved provider credentials in the
  matching `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `DEEPSEEK_API_KEY`, or
  `NVIDIA_API_KEY` entry. The application can use and report those keys but cannot change or remove
  them through Settings. Restart the web and worker services after an IT-managed key changes.

Do not send `server.env` to the vendor or include a key in a ticket or screenshot. After setup,
verify that Settings shows the intended provider as configured; it should expose only a masked
preview.

## What gets installed

Four Windows services, all managed by NSSM, all set to restart automatically:

| Service | What it does |
| --- | --- |
| `TermSheetWeb` | The application itself |
| `TermSheetWorker` | Background processing (document extraction, etc.) |
| `TermSheetProxy` | Caddy — terminates HTTPS on port 443 |
| `TermSheetBackup` | Scheduled database/document backups |

Logs for each are at `%ProgramData%\WinnerZone\TermSheet\logs\<ServiceName>.log` (and `.err.log` for
errors). If something looks wrong, this is the first place to look.

## Day-to-day operation

**Updates are automatic and require nothing from you.** The updater checks the manifest URL every
15 minutes by default (configurable at install time), and applies a new signed version if one's
available — downloading it, verifying its signature, taking a backup, running it alongside the
current version to confirm it's healthy, and only then switching over. If a newly-installed version
fails its own health check, the updater automatically switches back to the previous one; you don't
have to intervene, and the change happens in seconds.

To check status at any time:

```powershell
Get-Content "%ProgramData%\WinnerZone\TermSheet\updater-state.json" | ConvertFrom-Json
```

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

## If something goes wrong

**A service won't start.** Check its `.err.log` under `logs\`. Confirm PostgreSQL is reachable and
`server.env` is present and correctly restricted (only the service account and administrators should
be able to read it — the bootstrap sets this automatically, but a later manual edit can undo it).

**`preflight-connectivity.ps1` or `verify-release.ps1` starts failing after months of working fine.**
Most likely an outbound firewall or proxy rule changed. If it's specifically the public-key
fingerprint check that starts failing, stop and contact the vendor before doing anything else — that
specific failure is what a compromised update channel would look like.

**Disk space errors during an update.** The updater refuses to proceed unless there's room for the
new release plus enough headroom to roll back safely. Free up space; nothing will be left half
-installed by a refusal like this.

**You need to force an update check immediately**, rather than waiting for the next scheduled run —
ask the vendor for the exact command; it runs the same script the scheduled task does, so it's safe,
but the flags matter and depend on your install layout.

## Getting help

Contact your vendor with the version from `updater-state.json`'s `installedVersion`, the relevant
lines from the service's `.err.log`, and — if it's update-related — the full `updater-state.json`.
Don't send `server.env` or anything from the `config\` or `secrets\` folders; the vendor should never
need those, and they contain your credentials.
