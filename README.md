# gleec-wallet-builder

CI builder for [GLEECBTC/gleec-wallet](https://github.com/GLEECBTC/gleec-wallet)
desktop releases (Linux + Windows).

## Workflow: `Build Gleec Wallet macOS`

Separate macOS workflow under incremental development, running on the
`self-hosted`, `macOS`, `ARM64` runner. Manual dispatch and reruns are restricted
to the repository owner, from this repository's `main` branch.

Available `stage` inputs:

- `diagnostics` (default): inventory installed tools and the runner's GUI session.
  Missing tools are reported without failing the inventory. No checkout or signing
  secrets are used.
- `toolchain`: install Flutter 3.41.4 for arm64, download macOS/web artifacts,
  and require full Xcode with completed first launch and CocoaPods 1.16.2.
- `source`: run the toolchain checks, check out the wallet's requested `ref`,
  verify pinned recursive submodules, and apply `FIREBASE_PATCH`. The isolated
  checkout is removed after the run. `build_type` defaults to `debug`.
- `prepare`: run the source checks, fetch locked Flutter packages, install Pods,
  build web assets, and verify generated coin assets and the macOS KDF executable's
  arm64 support. Dependency version/source changes fail the run; checksum-only
  changes for local Flutter plugin podspecs are recorded and allowed after Dart
  packages have passed `pub get --enforce-lockfile`.
  The web build retries once for the transformer's explicit "Coin assets were
  updated" signal; all other failures stop preparation.
- `signing`: run preparation, import the Developer ID identity into a temporary
  keychain, validate/install the matching profile, and sign a small test binary
  without interactive prompts. The keychain search list is restored and temporary
  signing material is removed after the job.

- `build`: run signing preflight, compile `--release --flavor production`, and
  verify the app profile, entitlements, arm64 binaries, Developer ID signatures,
  timestamps and hardened runtime. Build service settings come from repository
  secrets; `debug`/`release` select the corresponding Matomo site ID.

The `macos-signing` GitHub environment is restricted to `main`. The signing stage
uses `MACOS_CERTIFICATE_P12_BASE64`, `MACOS_CERTIFICATE_PASSWORD`, and
`MACOS_PROVISIONING_PROFILE_BASE64` from that environment. Profile installation
targets Xcode 16+ at `~/Library/Developer/Xcode/UserData/Provisioning Profiles`;
existing profiles are preserved.

Environment setup uses shell commands, without third-party setup actions.
Flutter is downloaded directly from Google's official Flutter release archive.
The archive's SHA-256 and SDK git revision are pinned in the workflow and verified
before use. Each run installs into a temporary directory and removes it afterward.
When updating Flutter, update all three pins from the official
[macOS release manifest](https://storage.googleapis.com/flutter_infra_release/releases/releases_macos.json).

Xcode is installed and initialized once on the Mac by its administrator. The
workflow uses `/Applications/Xcode.app/Contents/Developer` by default; set the
repository variable `MACOS_XCODE_PATH` to use a different developer directory.
It does not change the system-wide Xcode selection or accept licenses.

Results appear in the run's job summary and logs. Notarization, DMG
packaging, and release publication will be added after the prerequisite stages
pass on the runner.

## Workflow: `Build Gleec Wallet Desktop`

Manually triggered (`workflow_dispatch`), **repo owner only** (enforced via
`github.actor == github.repository_owner` on every job).

### Inputs

| Input | Description | Default |
|---|---|---|
| `ref` | gleec-wallet commit SHA, branch, or tag to build | `main` |
| `build_type` | `debug` or `release` — selects the Matomo site ID and the release tag prefix | `debug` |

### What it does

1. Clones `GLEECBTC/gleec-wallet` (full history, recursive submodules) and
   checks out the requested `ref`.
2. Applies the Firebase production config patch from the `FIREBASE_PATCH`
   secret (`git apply -v`).
3. Installs Flutter 3.41.4 + platform toolchains (GTK/ninja/clang on Linux,
   MSBuild + Windows SDK 26100 on `windows-2022`).
4. `flutter pub get --enforce-lockfile`, then `flutter build web --no-pub`
   first — required so the build transformer downloads and registers coin
   icons, config files, and KDF assets.
5. `flutter build <platform> --no-pub --release` with `--dart-define`s taken
   from repository secrets (plus `COMMIT_HASH` and `BUILD_DATE`).
6. Packages the bundles as `gleec_wallet_linux_<id>.tar.gz` and
   `gleec_wallet_windows_<id>.zip`, where `<id>` is the sanitized tag name
   if `ref` is a tag, or the 7-char commit SHA if `ref` is a branch or
   commit.
7. On success of **both** platforms, creates a GitHub release in this repo
   tagged `debug_<id>` or `release_<id>` (debug builds are marked
   *pre-release*) and attaches the archives.

### Required repository secrets

| Secret | Purpose |
|---|---|
| `FIREBASE_PATCH` | Firebase production config patch, applied via `git apply` after checkout |
| `FEEDBACK_API_KEY` | Cloudflare feedback service API key |
| `FEEDBACK_PRODUCTION_URL` | Cloudflare feedback service URL |
| `TRELLO_BOARD_ID` | Trello board ID for the feedback service |
| `TRELLO_LIST_ID` | Trello list ID for the feedback service |
| `MATOMO_URL` | Matomo endpoint, e.g. `https://gwa.gleec.com/gwa.php` |

### Optional repository secrets

| Secret | Purpose | Fallback |
|---|---|---|
| `MATOMO_SITE_ID_DEBUG` | Matomo site ID for debug builds | `3` |
| `MATOMO_SITE_ID_RELEASE` | Matomo site ID for release builds | `2` |
| `GH_API_PUBLIC_READONLY_TOKEN` | GitHub PAT used by the build transformer to fetch public assets | ephemeral `github.token` of the workflow run |

Missing feedback/Trello/Matomo secrets do not fail the build — the
corresponding feature is simply compiled out (with a workflow warning).
