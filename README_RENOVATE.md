# Central Renovate & Releases

How the centralized Renovate setup works, how repositories onboard, how the shared
workflows are versioned and released, and how consumers pin them.

## Components

| File | Purpose |
| ---- | ------- |
| `.github/renovate.json5` | Central/global Renovate configuration applied to every swept repository (repository-local configs still apply on top) |
| `.github/renovate-repositories.json` | List of repositories (`owner/name`) covered by the nightly sweep |
| `.github/workflows/renovate.yml` | Reusable workflow (`workflow_call`) that validates the central config and runs Renovate for ONE target repository |
| `.github/workflows/renovate-schedule.yml` | Nightly sweeper: fans the repository list out over the reusable workflow; on PRs it validates the config/list plus a dry-run canary |
| `.github/workflows/release-please.yml` | Release automation: release PR from conventional commits → tag `vX.Y.Z` + release → re-pin floating major tag `vX` |
| `release-please-config.json`, `.release-please-manifest.json` | release-please configuration and the version baseline |

## How the Renovate setup works

- `renovate-schedule.yml` runs nightly and resolves the target list (or a
  `workflow_dispatch` override), then calls `renovate.yml` once per repository.
- `renovate.yml` first validates the central config (`renovate-config-validator`),
  then runs Renovate against the target repository.
- **Credentials:** write-capable runs use the `ODG_BOT` GitHub App
  (`vars.ODG_BOT_APP_ID` + `secrets.ODG_BOT_PRIVATE_KEY`), which is scoped to the
  **`renovate` environment** — jobs that need it declare that environment. Without
  App credentials (e.g. forks), runs degrade to a read-only dry-run
  (`RENOVATE_DRY_RUN=extract`); `dryRun: true` forces this even with credentials.
- **Commit authorship:** commits are made via the platform (`RENOVATE_PLATFORM_COMMIT`),
  so GitHub itself sets the App as author; no `gitAuthor` is configured centrally.
  PRs are opened under the App identity.
- **Config layering:** `.github/renovate.json5` is the global config for every
  target; the repository's own `renovate.json[5]` still applies on top.
  Onboarding PRs are disabled (`RENOVATE_ONBOARDING: false`), so a repository must
  bring its own Renovate config before it is swept.
- Renovate's repository cache lives in the caller repository's cache namespace,
  keyed per target.

## Onboarding a repository

1. Give the repository its own `renovate.json[5]` (extend presets / rules there;
   the central config applies additionally).
2. Add its `owner/name` to `.github/renovate-repositories.json` — the nightly
   sweep picks it up from the next run on.
3. For full (write) runs, install the `ODG_BOT` GitHub App on the repository with
   the permissions requested in `renovate.yml` (contents, issues, pull-requests,
   statuses, workflows, plus read on checks/vulnerability-alerts).
4. Optionally add a thin caller workflow for immediate push/PR-triggered runs
   (rebase/automerge), e.g.:

   ```yaml
   # .github/workflows/renovate.yml in the consuming repository
   name: Renovate
   on:
     workflow_dispatch: {}
   jobs:
     renovate:
       uses: open-component-model/.github/.github/workflows/renovate.yml@v1
       secrets: inherit
       with:
         repository: ${{ github.repository }}
   ```

## Versioning and releases

The shared workflows are released with
[release-please](https://github.com/googleapis/release-please), driven by
conventional commits on `main`:

| Commit message | Bump | Example result |
| -------------- | ---- | -------------- |
| `fix: ...` | patch | `v1.0.1` |
| `feat: ...` | minor | `v1.1.0` |
| `feat!: ...` / `BREAKING CHANGE: ...` | major | `v2.0.0` |
| `chore:`, `ci:`, `docs:`, ... | none | no release |

Flow:

1. Every push to `main` that changes renovate-related files runs `release-please.yml`, which maintains an accumulating
   **release PR** (`chore(main): release X.Y.Z`) with changelog and version bump.
2. **Merging the release PR is the release.** release-please creates tag `vX.Y.Z`
   and the GitHub release; the `move-major-tag` job in the same workflow then
   re-pins the floating major tag `vX` to the very same commit. (This is done via
   the action outputs because releases created by automation do not fire
   `release: published` events for a separate workflow.)
3. A major bump creates/moves the new `v2` tag; `v1` remains pinned to the last
   `v1.x.y`, so consumers on the old major line are unaffected.

Release notes follow the conventional commits between releases — write squash-merge
titles accordingly (`fix:`, `feat:`, ...). To override the proposed version (e.g.
for the initial release), commit with a `Release-As: X.Y.Z` footer:

```bash
git commit --allow-empty -m "chore: set initial release version" -m "Release-As: 1.1.0"
```

Credentials: the release jobs use the `ODG_BOT` App token from the `renovate`
environment (App tokens are unaffected by the "Allow GitHub Actions to create and
approve pull requests" repository setting). Without the App (forks), the workflow
falls back to `GITHUB_TOKEN`, which requires that setting to be enabled. Note that
bot-authored release PRs carry no `Signed-off-by` trailer — where DCO is enforced,
exempt the bot or amend the release PR with a sign-off before merging.

## Consuming the shared workflows

Reusable workflows are pinned by git ref, like actions:

```yaml
uses: open-component-model/.github/.github/workflows/renovate.yml@v1     # floating major (recommended)
uses: open-component-model/.github/.github/workflows/renovate.yml@v1.1.0 # exact release
```

- `@v1.X.Y` never moves — fully reproducible.
- `@v1` automatically follows the latest compatible release; a breaking workflow
  change ships as `@v2` and does not affect `@v1` consumers.

The central config consumed at runtime comes from the same repository, on the
same major line as the workflow pin (`centralConfig` input, default
`open-component-model/.github@v1`). No caller configuration is needed in the
common case; forks and development setups override it once via the repository
variable `RENOVATE_CENTRAL_CONFIG` (`owner/repo@ref`) or per run via the input —
a SHA-pinning caller can pass the same commit for bit-exact reproducibility.

To verify that a floating tag tracks its release:

```bash
git ls-remote --tags https://github.com/open-component-model/.github 'v1*'
# refs/tags/v1 and refs/tags/v1.X.Y must show the same commit SHA
```
