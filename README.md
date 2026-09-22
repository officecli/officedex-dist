# OfficeDex Distribution

This repository hosts the publicly-downloadable artifacts and the auto-update
manifest for [OfficeDex](https://github.com/officecli/officedex) — an AI
document generation desktop app. This `officedex-dist` repository exists so
the desktop app can fetch updates without authentication.

## Endpoints

| Resource | URL |
|---|---|
| Auto-update manifest (0.5.x / `stable`) | `https://raw.githubusercontent.com/officecli/officedex-dist/main/manifest.json` |
| Auto-update manifest (1.0) | `https://raw.githubusercontent.com/officecli/officedex-dist/main/channels/1.0/manifest.json` |
| Per-version binaries | GitHub Releases on `officecli/officedex` (`/releases/download/v<x.y.z>/`) |

`manifest.json` at the repository root is the 0.5.x production channel. Do not point it at 1.0.x while 0.5.x clients are still in the field. The 1.0 desktop builds bake the `channels/1.0/` URL.

The desktop app polls the manifest every 4 hours (and on window focus after
30 minutes of inactivity). The manifest format is documented in
`internal/appupdate/manager.go` in the source repository.

## Manifest schema

```json
{
  "version": "0.1.0",
  "notes": "Markdown-formatted release notes shown in the in-app banner.",
  "minSupportedVersion": "0.1.0",
  "mandatory": false,
  "publishedAt": "2026-05-25T00:00:00Z",
  "assets": {
    "darwin-arm64":     { "url": "...", "sha256": "...", "size": 12345 },
    "darwin-amd64":     { "url": "...", "sha256": "...", "size": 12345 },
    "darwin-universal": { "url": "...", "sha256": "...", "size": 12345 },
    "windows-amd64":    { "url": "...", "sha256": "...", "size": 12345 }
  }
}
```

- `mandatory: true` triggers a blocking force-update overlay in the client.
  Reserve this for security fixes or protocol breaks; use `minSupportedVersion`
  for routine end-of-life retirement instead.
- The client compares `version` against its own embedded version (semver) and
  only proposes an upgrade when greater.
- Asset URLs MUST point inside this repository so unauthenticated clients can
  download them. The Wails release workflow uploads zips here automatically.

## Directory layout

```
officedex-dist/
├── manifest.json                # 0.5.x production channel
├── archive/                     # Historical 0.5.x manifest snapshots
│   └── manifest-v0.5.43.json
└── channels/
    └── 1.0/
        ├── manifest.json        # develop/1.0 channel
        └── archive/
            └── manifest-v1.0.N.json
```

## How releases happen

Releases are driven from the source repo:

1.0 desktop builds are compiled and notarized on a maintainer machine
   (`officedex/scripts/build-mac-dmg.sh`), then published with
   `officedex/scripts/publish-update-channel.mjs --channel 1.0` into
   `channels/1.0/`. That script will refuse to modify the root `manifest.json`.
2. 0.5.x production still uses the historical tag → GitHub Release → root
   `manifest.json` path. Keep `/releases/latest` on that line by marking 1.0
   GitHub Releases as prerelease.

The atomic commit guarantees clients never see a manifest pointing at a
not-yet-uploaded binary.

## Manual override (testing / staging)

Set the `OFFICEDEX_UPDATE_MANIFEST_URL` environment variable on the client to
point at an alternate manifest — useful for staged rollouts, QA branches, or
canary builds. The client honours this on every check.

## Rollback

To revert to a previous version, restore an older `manifest.json` (either via
`git revert` or by copying from `archive/`) and push. Clients pick up the
change at the next polling tick (within 4 hours, or immediately on window
focus after the 30-minute threshold).
