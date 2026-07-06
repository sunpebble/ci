# sunpebble/ci

Shared GitHub Actions workflows for sunpebble iOS apps.

## Reusable workflows

| Workflow | Purpose |
|----------|---------|
| `ios-testflight.yml` | Simple sunpebble apps (automatic signing, `github.run_number` build) |
| `ios-testflight-fresh-pantry.yml` | Fresh Pantry: Secrets.plist, version.txt, simulator gate, Sentry dSYM |
| `ios-testflight-pathfinding.yml` | Pathfinding: manual signing, preinstalled profile, altool upload |

Call from an app repo after `release-please` creates a release:

```yaml
testflight:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v1
  with:
    xcode_project: Steady.xcodeproj
    scheme: Steady
    ipa_basename: Steady
  secrets: inherit
```

Pin `@v1` (or newer tag). Do not use `@main` in production callers.

## App configuration

See [`docs/app-matrix.json`](docs/app-matrix.json) for per-repo `scheme` / paths.

## GitHub settings

- Org/org repo must allow reusable workflows from `sunpebble/ci`.
- Secrets remain on each app repo (or org-level); this repo stores no credentials.

## Tags

| Tag | Notes |
|-----|-------|
| `v1` | Initial `ios-testflight` for simple iOS apps |
| `v1.1` | Adds `ios-testflight-fresh-pantry` and `ios-testflight-pathfinding` |