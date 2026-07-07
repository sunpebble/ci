# HarmonyOS AppGallery Reusable Workflow — Design

**Date:** 2026-07-07
**Status:** Approved (pending implementation)
**Owner:** sunpebble/ci

## Goal

Give every `*-harmony` app repository the same release-to-store loop that the
iOS apps already have, so a merged release-please release PR automatically
builds, signs, and submits the HarmonyOS build to Huawei AppGallery Connect
(AGC) for review.

## Non-goals

- Store metadata automation (screenshots, descriptions, privacy URLs). The
  `harmony-kit/README.md` explicitly defers this; out of scope here.
- Cross-platform matrix builds. One `.hap` per app per release.
- Phased rollout / A-B test configuration in AGC. Submit-for-review only.

## Reference (existing pattern)

`ci/.github/workflows/ios-testflight.yml` is the template this mirrors. It is a
`workflow_call` workflow with typed inputs, `defaults.run.working-directory`,
env-injected secrets, an official upload action, and an `if: always()` cleanup
step. The caller wires it behind release-please:

```yaml
testflight:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  uses: sunpebble/ci/.github/workflows/ios-testflight.yml@v2
  with: { ... }
  secrets: inherit
```

The HarmonyOS workflow replaces the App Store Connect upload with an AGC
Publishing API upload. Everything else (trigger, inputs shape, secrets
inheritance, cleanup) stays the same.

## Architecture

```
app repo (e.g. simmer-harmony)
  └── release-please-action  ──creates release──▶  tag v0.1.1
                                                       │
                          needs.release-please.outputs.release_created == 'true'
                                                       ▼
        sunpebble/ci/.github/workflows/harmony-appgallery.yml@v2
                                                       │
        ┌──────────────┬──────────────┬───────────────┴────────────┬──────────────┐
        ▼              ▼              ▼                            ▼              ▼
   setup Harmony   compute         hvigorw                  hap-sign-tool       AGC
   SDK + ohpm      versionCode     assembleHap              sign .hap           Publishing API
   + hvigor        write app.json5                          (p12/cer/p7b)       token→upload→submit
```

## Caller contract (per `*-harmony` repo)

```yaml
appgallery:
  needs: release-please
  if: needs.release-please.outputs.release_created == 'true'
  uses: sunpebble/ci/.github/workflows/harmony-appgallery.yml@v2
  with:
    app_subpath: apps/simmer            # repo-relative path to the DevEco app module
    bundle_name: com.sunpebble.simmer.harmony
    hap_basename: simmer-default-signed # basename of the .hap emitted by assembleHap
    agc_app_id: "1077xxxxxx"            # AppGallery app id (numeric, not secret)
  secrets: inherit
```

## Inputs

| Input | Type | Required | Default | Purpose |
|---|---|---|---|---|
| `working_directory` | string | no | `.` | Repo-relative root for `defaults.run.working-directory` |
| `app_subpath` | string | yes | — | Path under `working_directory` to the DevEco app module (contains `hvigorw`, `AppScope/app.json5`) |
| `bundle_name` | string | yes | — | `app.bundleName`, used for sanity check against `app.json5` |
| `hap_basename` | string | yes | — | Basename (no extension) of the signed `.hap` to upload |
| `sign_config_name` | string | no | `default` | Name of the signing config in the app's `build-profile.json5` |
| `submit_for_review` | boolean | no | `true` | If false, upload only and stop before the submit-phase call |
| `agc_app_id` | string | yes | — | AGC numeric app id; passed as input (not secret) |

## Secrets (per app repo, `secrets: inherit`)

The `ci` repo stores no credentials, matching the iOS workflow's contract.

| Secret | Purpose |
|---|---|
| `HARMONY_P12_BASE64` | Base64 of the `.p12` distribution keystore |
| `HARMONY_P12_PASSWORD` | Keystore password |
| `HARMONY_CER` | Distribution certificate (`.cer`) — base64 |
| `HARMONY_P7B` | Provisioning profile (`.p7b`) — base64 |
| `AGC_CLIENT_ID` | AGC Connect API client id |
| `AGC_CLIENT_SECRET` | AGC Connect API client secret |

`AGC_APP_ID` is an input, not a secret: it is numeric, app-specific, and not
sensitive.

## Pipeline steps

1. **Checkout** — `actions/checkout@v7` (matches iOS workflow).
2. **Setup HarmonyOS toolchain** — install `ohpm`, `hvigor` (Node wrapper), and
   `hap-sign-tool`. **Placeholder step.** No first-party GitHub Action exists;
   the step body documents three viable options (see Open Questions). The
   skeleton ships a documented `run:` block that fails loudly until the option
   is chosen.
3. **ohpm cache** — `actions/cache@v4` keyed on `oh-package-lock.json5` to
   speed reruns.
4. **Compute versionCode** — parse `major.minor.patch` from the release tag and
   compute `versionCode = major*1000000 + minor*1000 + patch`. (Existing
   `app.json5` ships `versionCode: 1000000` for `versionName: 0.1.0`; this
   formula keeps `1.0.0 → 1000000` consistent.) Patch the JSON5 file in place
   via a small Node one-liner (`json5` parse + stringify) and also update
   `versionName`.
5. **Build** — `node ./hvigor/hvigorw.js assembleHap --mode module -p product=default --no-daemon`.
   The `.hap` is emitted under `${app_subpath}/build/default/outputs/default/`;
   the workflow globs it and matches against `hap_basename`.
6. **Sign** — decode the three base64 secrets to `$RUNNER_TEMP/`, then invoke
   `hap-sign-tool sign-app` with the keystore, cert, and `.p7b`. The exact
   `--signAlg` / key alias arguments are a documented TODO pending the
   project's signing material (see Open Questions).
7. **AGC upload + submit** — four sequential `curl` calls against
   `https://connect-api.cloud.huawei.com/api`:
   1. `POST /oauth2/v1/token` → bearer token (client_id + client_secret)
   2. `GET /applications/v1/uploadurl` with `pkg_name=bundle_name` and a
      `suffix=hap` → upload URL + auth code
   3. `PUT` the signed `.hap` to the returned URL (with `Content-Type:
      multipart/form-data`)
   4. If `submit_for_review` is true: submit the new version for review via the
      App submitting API. The release/phase id required here is a documented
      TODO — the first AGC release must be created in the console to obtain it.
8. **Cleanup** — `if: always()` `rm -rf` of `$RUNNER_TEMP` decoded cert files.

## Key design choice: raw REST over community Action

Huawei does not publish a first-party, maintained GitHub Action for the AGC
Publishing API. Community actions exist but have spotty maintenance. Using raw
`curl` keeps the workflow:

- Transparent (every API call visible in the log)
- Dependency-free (no third-party action version drift)
- Debuggable (matches iOS's raw shell where no official action exists, e.g. the
  `ExportOptions.plist` step)

The cost is verbosity (~40 lines of curl + jq vs a single `uses:`). Acceptable
for a release path that runs at most a few times per week.

## versionCode strategy

`major*1000000 + minor*1000 + patch`. Rationale:

- Matches the seed value in `simmer-harmony/apps/simmer/AppScope/app.json5`
  (`versionCode: 1000000`, `versionName: "0.1.0"` → `1*1e6 + 0*1e3 + 0`).
- Monotonic under SemVer, sortable as an integer.
- Does not depend on `github.run_number`, so the same tag always produces the
  same `versionCode` (reproducible).

`patch` is capped at 999, `minor` at 999 — far beyond any realistic app.

## Open questions (resolved during first manual run)

These are *legitimate unknowns that require the user's signing material or AGC
console state to resolve*, not spec gaps. Each is flagged inline in the
skeleton with `# TODO(<ref>)`.

1. **HarmonyOS SDK install in CI — RESOLVED (Option A: re-hosted CLI).**
   The HarmonyOS Command Line Tools bundle `sdkmgr` + `ohpm` + `hvigor` +
   `hap-sign-tool`, but Huawei's Download Center sits behind a login wall —
   there is no anonymous public URL. Every public approach works around this.
   The community Action `harmonyos-dev/setup-harmonyos-sdk@0.2.1` is stale
   (last release Mar 2024, pre-NEXT, targets SDK `2.0.0.2`/API 9) and will not
   build HarmonyOS NEXT apps.

   **Chosen approach:** one-time re-host of the Linux Command Line Tools zip
   on a private `sunpebble/harmonyos-sdk` GitHub release. The workflow
   downloads the asset via the GitHub API using a PAT secret, unzips it, runs
   `sdkmgr install toolchains:<api>`, and caches the result.

   One-time setup (documented in `ci/README.md`):
   1. Sign in to Huawei Download Center, download
      `commandline-tools-linux-x86-6.0.x.zip`.
   2. Create a release on `sunpebble/harmonyos-sdk`, upload the zip as an
      asset.
   3. Create a PAT with `contents: read` on that repo; store it as the
      `SDK_DOWNLOAD_TOKEN` org/repo secret.

   Verified gotchas baked into the workflow:
   - `JAVA_TOOL_OPTIONS=-Duser.country=CN` is required or `sdkmgr`'s
     `getSdkList` call hangs on region detection.
   - CLI layout differs across versions: NEXT uses `command-line-tools/bin/sdkmgr`;
     older builds use `command-line-tools/sdkmanager/bin/sdkmgr`. The workflow
     probes both.
   - `hap-sign-tool.jar` is a jar (not a binary), located under
     `hwsdk/<api>/toolchains/`. The sign step locates it via `find` and invokes
     `java -jar`. Exact path must be verified on first run.
   - The sunpebble apps use hvigor 6.x with no `hvigorw` wrapper, so the build
     invokes `npx hvigor` after `npm install` (with `@ohos` npm registry set).

   Alternatives considered: (B) self-built Docker image pushed to GHCR —
   faster CI but more upfront work; (C) self-hosted runner — legally cleanest
   but adds ops overhead. Upgrade to B if per-run SDK install becomes slow.
2. **`hap-sign-tool` exact arguments — RESOLVED (validated with real material).**
   The tool is a jar at `<sdk>/default/openharmony/toolchains/lib/hap-sign-tool.jar`,
   invoked via `java -jar`. Its `sign-app` subcommand uses single-dash camelCase
   flags (NOT `--key-store` style), requires `-mode localSign`, and needs
   `-compatibleVersion` for hap inputs. **Validated end-to-end on
   `simmer-harmony`** with a real AGC-issued keystore + cert chain + release
   profile — produced `entry-default-signed.hap`, `Sign Hap success!`:
   ```
   sign-app -mode localSign \
     -keyAlias harmony -keyPwd <pwd> \
     -appCertFile release.cer -profileFile release.p7b \
     -inFile entry-default-unsigned.hap -outFile entry-default-signed.hap \
     -signAlg SHA256withECDSA \
     -keystoreFile release.p12 -keystorePwd <pwd> \
     -compatibleVersion 23 -signCode 1
   ```
   Notes: `release.cer` from AGC is a **3-cert PEM chain** (Root CA G2 →
   Developer Relations CA G2 → app cert); `openssl x509` only reads the first
   cert so the chain must be passed whole. `-compatibleVersion 23` matches the
   apps' `compatibleSdkVersion: 6.1.0(23)`. The keystore's `-keyAlias`
   (`harmony` here) is a required `key_alias` workflow input.
3. **AGC release phase id for submit.** Create the first release in the AGC
   console to obtain the `release_id`; the workflow can then list releases and
   target it. Or set `submit_for_review: false` initially and submit manually.

## Local validation (build step)

The build half of the pipeline was validated end-to-end on the developer Mac
against `simmer-harmony`, using the DevEco-bundled toolchain:

- `ohpm install --all` → ok (ohpm 6.1.1.830)
- `hvigorw assembleHap --mode module -p product=default -p buildMode=release --no-daemon` → **BUILD SUCCESSFUL**
- Output: `apps/simmer/entry/build/default/outputs/default/entry-default-unsigned.hap`

Findings folded back into the workflow:
- The hap is emitted under `<app>/<module>/build/...` (an extra `entry/` layer
  vs. the first draft), with filename `<module>-default-unsigned.hap`. The
  resolve step now constructs the path from a `module` input rather than a
  guessed `hap_basename`.
- `signingConfigs: []` in the project means hvigor skips signing and emits an
  unsigned hap — exactly what the standalone `sign-app` step consumes.
- The version-write sed regex patches `versionCode`/`versionName` correctly
  under GNU sed (Ubuntu CI). macOS BSD `sed -i` needs `-i ''`; CI is unaffected.

Signing was exercised locally with real AGC material (see above). AGC upload
was exercised against the live API; auth and the upload-url endpoint are
validated. The remaining blocker is app state, not code.

## AGC upload validation (live API)

Driven against the real AGC endpoint with the user's credentials:

- **Auth = team-level API client, `client_credentials` grant.** Token endpoint
  `POST https://connect-api.cloud.huawei.com/api/oauth2/v1/token` with
  `grant_type=client_credentials` + `client_id` + `client_secret` returns an
  `access_token` (48h TTL). Validated.
- **The API client MUST be team-level (项目 = N/A).** A project-level client
  (`type: project_client_id`, JSON contains `project_id`) returns HTTP **403**
  on every `/publish/v2/*` call — this is documented AGC behaviour. Validated
  by creating both and comparing (project → 403, team → works).
- **Publish endpoints need a `client_id` request header** alongside the
  `Authorization: Bearer` header. Without it, the call returns 401
  `client token auth failed`. (Source: the v1 docs omit this; the v2 effective
  behaviour requires it.)
- **`upload-url` params:** `appId` (capital I) + `suffix` = the file extension
  (`hap`), constrained to 1–6 chars. Not `suffix=release` (that overruns the
  length limit) and not lowercase `appid`.
- **Upload + attach flow** (multipart POST → PUT app-file-info): structured per
  the documented flow but **not yet exercised end-to-end** — `upload-url`
  returns `get file authCode failed` because the app has no draft release yet.
  This is open-question-3, now concretely confirmed.

## Open question 3 (confirmed in practice)

`upload-url` returns `{"code":204144645,"msg":"...get file authCode failed"}`
when the app has zero versions (`GET /publish/v2/app-versions` returns empty).
**The first release must be created in the AGC console** (upload the signed
hap, fill metadata, submit). After a draft release exists, the API can drive
subsequent releases. The workflow's submit step stays a TODO until that first
manual release is done and the submit endpoint is wired.

## Validation plan

Because every step beyond `checkout` depends on assets the user must supply,
the skeleton is validated incrementally:

1. **Dry job** — push the workflow, call it from a branch with a dummy release.
   Step 2 (SDK setup) must succeed before any later step is trusted.
2. **Build only** — temporarily set `submit_for_review: false` and skip upload;
   confirm `assembleHap` emits a `.hap`.
3. **Sign only** — confirm `hap-sign-tool` produces a signed `.hap` with the
   project's real keystore.
4. **Upload only** — confirm AGC token + upload URL round-trip succeeds.
5. **Full loop** — flip `submit_for_review: true` and confirm AGC shows the
   release in "under review".

This mirrors `harmony-kit/README.md`'s stated sequence: prove one app builds
in DevEco before adding release automation.

## Files

- **New:** `ci/.github/workflows/harmony-appgallery.yml` (~210 lines). SDK
  toolchain is wired via the re-hosted-CLI approach; two documented TODOs remain
  (signing args, AGC submit phase).
- **Update:** `ci/README.md` — workflow table row + HarmonyOS caller example +
  one-time SDK setup section.
