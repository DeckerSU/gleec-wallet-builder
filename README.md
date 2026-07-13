# gleec-wallet-builder

CI builder for [GLEECBTC/gleec-wallet](https://github.com/GLEECBTC/gleec-wallet)
desktop releases (Linux + Windows).

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
