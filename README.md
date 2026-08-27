# Snappy update feed

The signed data Snappy reads to judge whether a PC's BIOS, drivers and
Windows build are current.

Everything here is generated and pushed by CI. Nothing in this repository is
edited by hand, so pull requests against it cannot be merged - the next
scheduled run would overwrite them.

## Files

| file | what it holds |
| --- | --- |
| `feed/updates.json` | the BIOS, driver and Windows versions the app compares against |
| `feed/updates.sig` | ECDSA P-256 signature over `updates.json`, base64 |
| `feed/public-key.txt` | the signing key's public half, SubjectPublicKeyInfo, base64 |
| `feed/app.json` | the newest published Snappy version |

## Verifying a copy

The app carries its own copy of the public key and refuses any feed whose
signature does not verify against it, so a tampered copy of this data cannot
be fed to it. The same check by hand, in PowerShell 7 (Windows PowerShell's
ECDsaCng cannot import a SubjectPublicKeyInfo):

```powershell
$base = 'https://raw.githubusercontent.com/MattNorthOfficial/snappy-update-feed-public/main/feed'
Invoke-WebRequest -UseBasicParsing "$base/updates.json" -OutFile updates.json
$signature = (Invoke-WebRequest -UseBasicParsing "$base/updates.sig").Content.Trim()
$publicKey = (Invoke-WebRequest -UseBasicParsing "$base/public-key.txt").Content.Trim()

$ecdsa = [System.Security.Cryptography.ECDsa]::Create()
$read = 0
$ecdsa.ImportSubjectPublicKeyInfo([Convert]::FromBase64String($publicKey), [ref] $read)
$ecdsa.VerifyData(
    [System.IO.File]::ReadAllBytes("$PWD/updates.json"),
    [Convert]::FromBase64String($signature),
    [System.Security.Cryptography.HashAlgorithmName]::SHA256)
```

`True` means the file is exactly what was signed. The signature covers the
bytes of `updates.json`, so fetch it as bytes rather than as text - a copy
that has been through newline translation will not verify.

## Where the data comes from

Vendor sources swept on a schedule: AMD's and NVIDIA's driver releases, Intel
driver branches, Microsoft's Windows release information, Dell's update
catalog, and the BIOS listings of the four desktop board vendors. Each section
carries its own freshness stamp in `updates.json`, and a section whose source
cannot be reached keeps its last known values rather than disappearing.

The scrapers that produce this data, and the signing key, live elsewhere. Only
their output lands here.
