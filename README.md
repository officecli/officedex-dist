# OfficeDex Distribution

This repository hosts the publicly-downloadable artifacts and the auto-update
manifest for [OfficeDex](https://github.com/officecli/officedex) — an AI
document generation desktop app. This `officedex-dist` repository exists so
the desktop app can fetch updates without authentication.

## Endpoints

| Resource | URL |
|---|---|
| Auto-update manifest | `https://raw.githubusercontent.com/officecli/officedex-dist/main/manifest.json` |
| Per-version binaries | `https://raw.githubusercontent.com/officecli/officedex-dist/main/releases/v<x.y.z>/` |

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
├── manifest.json                # Single source of truth for current release
├── releases/
│   ├── v0.1.0/
│   │   ├── OfficeDex-v0.1.0-darwin-universal.zip
│   │   └── OfficeDex-v0.1.0-windows-amd64.zip
│   └── v0.x.y/...
└── archive/                     # Historical manifest snapshots (optional)
    └── manifest-v0.1.0.json
```

## How releases happen

Releases are driven from the source repo:

1. Maintainer bumps version in `app.go`, `package.json`, `wails.json` and tags
   `vX.Y.Z` in `officecli/officedex`.
2. The `Release` GitHub Actions workflow builds both platforms and publishes
   a GitHub Release with the zipped artifacts.
3. The workflow's `Sync to officedex-dist` step then clones this repo, copies
   the binaries into `releases/vX.Y.Z/`, regenerates `manifest.json`, and
   pushes a single commit. This step uses a fine-grained PAT stored as the
   `DIST_DEPLOY_TOKEN` secret.

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
