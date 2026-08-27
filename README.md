# Snappy update feed

The signed data Snappy reads to judge whether a PC's BIOS, drivers and
Windows build are current.

Everything here is generated and pushed by CI. Nothing in this repository is
edited by hand, so pull requests against it cannot be merged - the next
scheduled run would overwrite them.

- [Files](#files)
- [Verifying a copy](#verifying-a-copy)
- [Feed format](#feed-format)
- [Freshness](#freshness)
- [Where the data comes from](#where-the-data-comes-from)

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

The signature is IEEE P1363 format (a fixed-width `r || s` pair), not DER.
Verifiers that default to DER, OpenSSL among them, need telling.

The public key's SHA-256 fingerprint, for comparison against a copy obtained
some other way:

```
5748dbf69e5a3fda65628b30aef1ea28972532285a296ccf491b0d6d39767f9d
```

The app pins this key inside its signed package rather than trusting whatever
sits beside the feed, so a key change requires an app release, and a fetched
key alone can never move a machine's trust root.

## Feed format

```jsonc
{
  "schemaVersion": 1,
  "updated": "2026-08-05T13:06:09Z",
  "freshness":   { "amd.windows": "...", "motherboards.gigabyte": "..." },
  "communitySources": { "...": { "repository": "...", "commit": "..." } },

  "amd":           { "windows": { "current": {...}, "rdna1-2": {...}, "polaris-vega": {...} },
                     "chipset": { "revision": "...", "date": "...", "components": {...} } },
  "nvidia":        { "gameReady": "610.88", "url": "...", "download": "..." },
  "intel":         { "chipset": {...}, "arc": {...}, "xe": {...},
                     "rst20": {...}, "rst21": {...}, "chipsetInf": {...} },
  "windowsBuilds": { "25H2": { "build": "...", "date": "...", "kb": "...",
                               "eosHome": "...", "eosEnterprise": "..." } },
  "windows10":     { "22H2": { "eosHome": "...", "eosEnterprise": "..." } },
  "motherboards":  { "B650 AORUS ELITE": { "bios": "F41", "date": "...",
                                           "url": "...", "vendor": "gigabyte" } },
  "motherboardConflicts": { "...": { "msi": {...}, "gigabyte": {...} } },
  "dell":          { "0A5C": { "bios": "1.15.0", "date": "...", "url": "..." } }
}
```

- `schemaVersion` changes only for a consumer-facing breaking change.
  Additive fields stay within the current version, so a consumer that ignores
  unknown keys will not be broken by a new section appearing.
- `amd.windows` splits into the mainline release (`current`) and the
  maintenance branches AMD keeps for older GPU generations (`rdna1-2`,
  `polaris-vega`). Each carries `adrenalin`, the marketing version AMD's site
  shows, and `driverStore`, the version WMI and Device Manager report - which
  is what lets a client match an installed driver to the right branch.
- `intel` holds the chipset INF utility and the two graphics-driver branches
  Intel has maintained since its 2025 split: `arc` (Arc cards and Core Ultra
  iGPUs) and `xe` (the legacy package for 11th-14th gen Iris Xe / UHD).
- `windowsBuilds` maps each Windows 11 version to its latest *required*
  build. Patch Tuesday (B) and out-of-band releases count; optional D-week
  previews do not, so a fully patched machine is never flagged as outdated.
- `motherboards` is keyed by the clean marketing name, without the
  "(MS-7E51)"-style suffix WMI appends, and holds the latest non-beta BIOS
  with a link to the vendor's own page. Where several board revisions share
  one marketing name the entry is marked `revisionAmbiguous`; consumers
  should withhold a verdict rather than send every revision to one file.
- `motherboardConflicts` records models whose name is claimed by more than
  one vendor, with each vendor's answer kept separate.
- `dell` is keyed by the 4-hex system id every Dell and Alienware reports as
  its SKU, from Dell's own update catalog.
- `download`, where present, is the vendor's own installer URL, so a client
  can offer the file straight from the vendor's CDN.
- `communitySources` records the immutable commits behind the Intel INF and
  NVIDIA product-ID maps. Nothing here follows a moving upstream branch.

## Freshness

`updated` is when the file was written. `freshness` is the more useful field:
it stamps each section as that section's own check finishes, so a section that
could not be refreshed keeps its previous timestamp instead of inheriting a
misleadingly recent one.

That distinction matters because **a section whose source cannot be reached
keeps its last known values rather than disappearing**. Consumers should read
`freshness` to tell a newly written section from one carried forward, and
treat a section's age as a property of that section rather than of the file.

Driver, graphics and Windows sections refresh several times a day. The
motherboard sections are swept on a slower cycle, so a board entry is
routinely a day or two old by design; BIOS releases are rare enough that this
costs nothing in practice.

## Where the data comes from

Vendor sources swept on a schedule: AMD's and NVIDIA's driver releases, Intel
driver branches, Microsoft's Windows release information, Dell's update
catalog, and the BIOS listings of the four desktop board vendors. Values are
taken from each vendor's own published pages or catalogs - nothing here is
community-submitted, and nothing is inferred where a vendor is ambiguous.

The scrapers that produce this data, and the signing key, live elsewhere. Only
their output lands here.
