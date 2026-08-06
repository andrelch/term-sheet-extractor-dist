# Term Sheet Extractor — distribution channel

This repository distributes signed, runtime-only releases of the Term Sheet Extractor server.
It contains **no source code** and never will. Everything of substance lives in the
[Releases](../../releases) of this repository; the repository tree itself holds only this file.

Releases are published automatically by the release workflow in the private source repository.
Do not upload assets here by hand.

## For operators

Installed servers poll one fixed URL, which never changes across releases:

```text
https://github.com/andrelch/term-sheet-extractor-dist/releases/download/server-production/production.json
```

No credential is required to read it. Do not place a GitHub token in updater configuration —
these assets are public and there is nothing for a token to authorise.

## Release layout

| Tag | Mutability | Contents |
| --- | --- | --- |
| `server-vX.Y.Z` | Immutable | The versioned server tarball, bootstrap zip, their SHA-256 checksums, the signed manifest, and the release signing public key |
| `server-production` | Mutable | Only `production.json` and `release-signing-public.pem`. This is the production channel every updater polls |

## Verifying a release

Every manifest is signed with the project's release key. The updater verifies the signature
before applying anything, so an unsigned or tampered asset published here cannot be installed.

Confirm out of band — against the fingerprint supplied by the vendor through a channel other than
this repository — that the public key you hold is the expected one:

```powershell
openssl pkey -pubin -in release-signing-public.pem -outform DER | openssl dgst -sha256
```

Then verify a downloaded artifact against its published checksum:

```powershell
Get-FileHash .\term-sheet-server-X.Y.Z.tar.gz -Algorithm SHA256
```
