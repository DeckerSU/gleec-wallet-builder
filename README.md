# gleec-wallet-builder

CI builder for [GLEECBTC/gleec-wallet](https://github.com/GLEECBTC/gleec-wallet)
desktop releases (Linux, Windows and macOS).

## Workflow: `Build Gleec Wallet macOS`

Separate macOS workflow running on the
`self-hosted`, `macOS`, `ARM64` runner. Manual dispatch and reruns are restricted
to the repository owner, from this repository's `main` branch.

Available `stage` inputs:

- `diagnostics` (default): inventory installed tools and the runner's GUI session.
  Missing tools are reported without failing the inventory. No checkout or signing
  secrets are used.
- `toolchain`: install Flutter 3.41.4 for arm64, download macOS/web artifacts,
  and require full Xcode with completed first launch and CocoaPods 1.16.2.
- `source`: run the toolchain checks, check out the wallet's requested `ref`,
  verify pinned recursive submodules, apply `FIREBASE_PATCH`, and backport the
  SDK temporary-directory fix when missing. The isolated
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

- `sandbox-check`: compare small signed helper programs with legacy signing,
  hardened runtime, and sandbox inheritance. This diagnostic does not build or
  launch the wallet; its temporary signing material is removed afterward.

- `build`: run signing preflight, compile `--release --flavor production`, and
  verify the app profile, entitlements, arm64 binaries, Developer ID signatures,
  timestamps and hardened runtime. Build service settings come from repository
  secrets; `debug`/`release` select the corresponding Matomo site ID.
  For the legacy KDF helper signing command in wallet 0.9.7, the temporary
  checkout adds `--options runtime --timestamp` to the existing Xcode build
  phase. Signing still happens inside Xcode, before framework embedding; app
  entitlements are not passed to KDF. Final binary checks remain mandatory.
  This temporary change awaits [wallet PR #3541](https://github.com/GLEECBTC/gleec-wallet/pull/3541).

- `notary-auth`: verify the toolchain/signing setup and validate Apple
  notarization credentials without rebuilding the wallet. The profile
  `AC_NOTARY_GLEEC` is created in the temporary keychain and removed afterward.

- `notarize`: build and verify the app, submit a ZIP to Apple, require
  `Accepted`, staple the ticket, and validate it with stapler and Gatekeeper.
  The submission ID is recorded in the job summary before waiting. A timeout
  queries the same submission and never automatically uploads another copy.

- `dmg`: first require a console session and Finder automation, then build and
  notarize the app and run the wallet's `contrib/make-dmg.sh`. The completed image
  is mounted readonly to verify its app signature, stapled ticket, Gatekeeper
  assessment and Applications shortcut. The DMG, SHA-256 and build manifest are
  uploaded as a 14-day artifact. An already-mounted image with the same volume
  name causes a failure; the workflow never forcibly detaches that volume.

- `dmg-check`: test Finder window layout on a small temporary disk image without
  installing Flutter, building the wallet or accessing signing secrets. Use this
  to check GUI permissions before a full build.

- `publish`: run the full DMG build, then publish from a separate Ubuntu job with
  release write access. It verifies the artifact manifest and checksum, uploads
  the DMG plus its `.sha256` and `.json` metadata, and downloads the published
  DMG to verify its checksum again. Existing release notes and other platform
  assets are preserved. Only the explicitly named macOS assets can be replaced.

`dmg` and `publish` include the small Finder layout probe before installing Flutter.
Both `debug` and `release` use Flutter release compilation and Developer ID signing;
the selected service configuration also determines `debug_<safe_id>` (prerelease)
or `release_<safe_id>` (regular release). As in the desktop workflow, `safe_id`
is the sanitized wallet tag when `ref` names a tag, otherwise its short commit SHA.
The DMG filename always uses the wallet commit: `gleecdex_macos_<short_sha>.dmg`.

DMG layout requires a logged-in GUI session for the runner user. When macOS asks
whether Terminal may control Finder, click **Allow** on the Mac. For a runner
started from Terminal, the permission is under **System Settings → Privacy &
Security → Automation → Terminal → Finder**. Run `dmg-check` once to confirm the
permission; the workflow cannot approve the macOS dialog itself.

Notarization JSON reports are also retained as artifacts, including on failures
after submission, so submission IDs remain available for diagnosis.

The separate SDK compatibility step embeds the reviewed patch from
[SDK PR #395](https://github.com/GLEECBTC/komodo-defi-sdk-flutter/pull/395), pinned
at `b08501c0900086a9f6d562700bb6df785ffb5894`. It creates the temporary directory
before `createTemp`, fixing KDF startup on a fresh sandboxed macOS installation.
The step checks whether the patch is already present before applying it; an
unrecognized source version fails with an explicit error. It never downloads
the current PR diff during a build. The summary and DMG manifest record whether
the patch was applied or already present.

Remove this SDK backport and the temporary helper-signing adjustment once their
respective PRs are merged **and the wallet refs being built include the fixes**.
Old tags such as `0.9.7` keep their original SDK commit and still need the backport.

The `macos-signing` GitHub environment is restricted to `main`. The signing stage
uses `MACOS_CERTIFICATE_P12_BASE64`, `MACOS_CERTIFICATE_PASSWORD`, and
`MACOS_PROVISIONING_PROFILE_BASE64` from that environment. Profile installation
targets Xcode 16+ at `~/Library/Developer/Xcode/UserData/Provisioning Profiles`;
existing profiles are preserved. Notarization authentication uses environment
secrets `APPLE_ID` and `APPLE_APP_SPECIFIC_PASSWORD` with team `B52ZCS7TMQ`.

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

Results appear in the run's job summary, logs and artifacts. For a complete build
and publication, choose `stage=publish`, the wallet `ref`, and `build_type`:

```bash
gh workflow run build-macos.yml --repo DeckerSU/gleec-wallet-builder --ref main \
  -f stage=publish -f ref=0.9.7 -f build_type=release
```

If only publication fails, use **Re-run failed jobs** while the DMG artifact is
still retained (14 days). The artifact name is stable across attempts of that
run, so publication can retry without rebuilding or resubmitting to Apple.

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
