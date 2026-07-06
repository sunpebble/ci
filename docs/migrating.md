# Migrating an app to sunpebble/ci

1. Ensure `release-please` job stays in `.github/workflows/release.yml`.
2. Replace the `testflight` job body with `uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v1`.
3. Set `with:` from `docs/app-matrix.json`.
4. Add `secrets: inherit`.
5. Remove duplicate steps (Xcode select, xcodegen, cert import, archive, export, upload).
6. For `fresh-pantry`, keep `quality` job and Secrets.plist generation in the caller; use `working_directory: apps/ios`.

## fresh-pantry

Not yet wired to reusable workflow only — requires `Secrets.plist` before `xcodegen`. Use composite action (future) or a dedicated `prepare` job in the app repo.

## pathfinding

Uses `apps/ios/Pathfinding` as `working_directory`. Align `release-please` output names when consolidating workflows.