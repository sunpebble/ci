# Migrating an app to sunpebble/ci

1. Ensure `release-please` job stays in `.github/workflows/release.yml`.
2. Replace the `testflight` job body with `uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v1`.
3. Set `with:` from `docs/app-matrix.json`.
4. Add `secrets: inherit`.
5. Remove duplicate steps (Xcode select, xcodegen, cert import, archive, export, upload).
6. For `fresh-pantry`, keep `quality` job and Secrets.plist generation in the caller; use `working_directory: apps/ios`.

## fresh-pantry

`uses: sunpebble/ci/.github/workflows/ios-testflight-fresh-pantry.yml@v1.1` with `secrets: inherit`. Keep `quality` + `release-please` in the app repo.

## pathfinding

`uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v1.2` with `working_directory: apps/ios/Pathfinding`, `secrets: inherit` (org `DIST_CERT_P12` / `DIST_CERT_PASSWORD`). Release signing is Automatic in `project.yml` (same as other sunpebble iOS apps).