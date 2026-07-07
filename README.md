# sunpebble/ci

Shared GitHub Actions workflows for sunpebble iOS and HarmonyOS apps.

## Reusable workflows

| Workflow | Purpose |
|----------|---------|
| `ios-testflight.yml` | Simple sunpebble apps (automatic signing, `github.run_number` build) |
| `ios-testflight-fresh-pantry.yml` | Fresh Pantry: Secrets.plist, version.txt, simulator gate, Sentry dSYM |
| `ios-testflight-pathfinding.yml` | Pathfinding: package.json version, Pathfinding-Release, ASC automatic signing |
| `harmony-appgallery.yml` | HarmonyOS apps: version bump, build, sign, upload to AppGallery Connect (toolchain wired via re-hosted CLI; signing args + AGC submit still TODO) |

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

### HarmonyOS

```yaml
appgallery:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  uses: sunpebble/ci/.github/workflows/harmony-appgallery.yml@v2
  with:
    app_subpath: apps/simmer
    bundle_name: com.sunpebble.simmer.harmony
    module: entry
    key_alias: simmer-release-key      # the -keyAlias used when the .p12 was generated
    compatible_version: "23"
    agc_app_id: "1077xxxxxx"
    version: ${{ needs.release-please.outputs.version }}
  secrets: inherit
```

Per-app secrets: `HARMONY_P12_BASE64`, `HARMONY_P12_PASSWORD`, `HARMONY_CER_BASE64`, `HARMONY_P7B_BASE64`, `AGC_CLIENT_ID`, `AGC_CLIENT_SECRET`. Org-wide secret: `SDK_DOWNLOAD_TOKEN`.

> `AGC_CLIENT_ID` / `AGC_CLIENT_SECRET` must come from a **team-level (项目 = N/A)** API client — project-level clients get HTTP 403 on `/publish/v2/*`.

See [`docs/superpowers/specs/2026-07-07-harmony-appgallery-workflow-design.md`](docs/superpowers/specs/2026-07-07-harmony-appgallery-workflow-design.md) — build + signing + AGC auth/upload-url are validated; one open TODO remains (first release must be created in the AGC console before API submit works).

### HarmonyOS SDK one-time setup

Huawei's Command Line Tools (bundles `sdkmgr` + `ohpm` + `hvigor` + `hap-sign-tool`) are behind a login wall, so we re-host the Linux zip:

1. Sign in to the [Huawei Download Center](https://developer.huawei.com/consumer/cn/download/), download `commandline-tools-linux-x86-<ver>.zip` (NEXT-era, e.g. 6.0.x).
2. Create a release on `sunpebble/harmonyos-sdk` (create the repo if absent), tag it `commandline-tools-<ver>`, upload the zip as an asset.
3. Create a fine-grained PAT with `contents: read` on `sunpebble/harmonyos-sdk`; store it as the `SDK_DOWNLOAD_TOKEN` secret (org-level, shared by all `*-harmony` repos).
4. Update the caller's `sdk_release_tag` / `cli_zip_name` inputs if you used a different version.

## App configuration

See [`docs/app-matrix.json`](docs/app-matrix.json) for per-repo `scheme` / paths.

## GitHub settings

- Org/org repo must allow reusable workflows from `sunpebble/ci`.
- Secrets remain on each app repo (or org-level); this repo stores no credentials.

## Tags

| Tag | Notes |
|-----|-------|
| `v2` | Current major: extended `ios-testflight` + `ios-testflight-fresh-pantry` |