# Migrating an app to sunpebble/ci

1. Ensure `release-please` job stays in `.github/workflows/release.yml`.
2. Replace the `testflight` job body with `uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v2`.
3. Set `with:` from `docs/app-matrix.json`.
4. Add `secrets: inherit`.
5. Remove duplicate steps (Xcode select, xcodegen, cert import, archive, export, upload).

## fresh-pantry

`uses: sunpebble/ci/.github/workflows/ios-testflight-fresh-pantry.yml@v2` with `secrets: inherit`. Keep `quality` + `release-please` in the app repo.

## pathfinding

`uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v2` with `working_directory: apps/ios/Pathfinding`, `secrets: inherit` (org `DIST_CERT_*`).