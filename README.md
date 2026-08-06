# Term Sheet Extractor — distribution channel

This public repository distributes signed, runtime-only Term Sheet Extractor server releases. It contains no application source.

Versioned assets are published automatically by the private source repository. Do not upload or replace release assets by hand.

## For operators

Installed servers poll this fixed anonymous HTTPS URL:

```text
https://raw.githubusercontent.com/andrelch/term-sheet-extractor-dist/main/production.json
```

The `main` branch holds the signed `production.json` channel manifest and the release public key. Each channel promotion is an atomic commit with Git history. Application artifacts and bootstrap packages live in immutable GitHub Releases tagged `server-vX.Y.Z`.

Never put a GitHub token in updater configuration. Downloads are public, and authenticity comes from the signed manifest plus the separately verified, pinned public key—not from repository access.

## Initial installation

Download `term-sheet-bootstrap-X.Y.Z.zip` from the intended immutable release. Before running any PowerShell, compare the key fingerprint in its `README.txt` with the value supplied out of band by the vendor:

```powershell
(Get-FileHash .\release-signing-public.pem -Algorithm SHA256).Hash.ToLowerInvariant()
```

Do not establish trust by comparing only with the copy in this repository; a separate channel is required. The bootstrap package includes read-only connectivity and end-to-end verification scripts and refuses a signed initial release older than its own version.

## Release layout

| Location | Mutability | Contents |
| --- | --- | --- |
| `server-vX.Y.Z` GitHub Release | Immutable after publication | Server tarball, bootstrap zip, SHA-256 files, signed manifest, public key |
| `main:production.json` | Atomic, history-retained branch update | Signed production channel manifest polled by installed servers |
| `main:release-signing-public.pem` | Pinned once; workflow refuses replacement | Convenience copy only; verify its fingerprint out of band |

The updater verifies the manifest signature, updater compatibility, artifact name, exact size, SHA-256, archive paths, and internal release marker before activation.