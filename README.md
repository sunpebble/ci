# sunpebble/ci

Shared GitHub Actions workflows for sunpebble iOS apps.

## Reusable workflows

| Workflow | Purpose |
|----------|---------|
| `ios-testflight.yml` | Simple sunpebble apps (automatic signing, `github.run_number` build) |
| `ios-testflight-fresh-pantry.yml` | Fresh Pantry: Secrets.plist, version.txt, simulator gate, Sentry dSYM |
| `ios-testflight-pathfinding.yml` | Pathfinding: package.json version, Pathfinding-Release, ASC automatic signing |

Call from an app repo after `release-please` creates a release:

```yaml
testflight:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v2
  with:
    xcode_project: Steady.xcodeproj
    scheme: Steady
    ipa_basename: Steady
  secrets: inherit
```

Pin **`@v2`** (major tag only). Do not use `@main` or semver tags like `@v1.2`.

## App configuration

See [`docs/app-matrix.json`](docs/app-matrix.json) for per-repo `scheme` / paths.

## GitHub settings

- Org/org repo must allow reusable workflows from `sunpebble/ci`.
- Secrets remain on each app repo (or org-level); this repo stores no credentials.

## Tags

| Tag | Notes |
|-----|-------|
| `v2` | Current major: extended `ios-testflight` + `ios-testflight-fresh-pantry` |