# Central Renovate & Releases

How the centralized Renovate setup works, how repositories onboard, how the shared
workflows are versioned and released, and how consumers pin them.

## Components

| File | Purpose |
| ---- | ------- |
| `.github/renovate-central.json5` | Central/global Renovate configuration applied to every repository that calls `renovate.yml` - nightly sweep and self-hosted workflows alike (repository-local configs still apply on top) |
| `.github/renovate-repositories.json` | List of repositories (`owner/name`) covered by the nightly sweep |
| `.github/workflows/renovate.yml` | Reusable workflow (`workflow_call`) that validates the central config and runs Renovate for ONE target repository |
| `.github/workflows/renovate-schedule.yml` | Nightly sweeper: fans the repository list out over the reusable workflow; on PRs it validates the config/list plus a dry-run canary |
| `.github/workflows/release-please.yml` | Release automation: release PR from conventional commits → tag `vX.Y.Z` + release → re-pin floating major tag `vX` |
| `release-please-config.json`, `.release-please-manifest.json` | release-please configuration and the version baseline |

## Central Renovate config (`.github/renovate-central.json5`)

This file is checked out at runtime by `renovate.yml` (via the `centralConfig`
input, defaulting to `open-component-model/.github@v1`) and passed to Renovate
as `RENOVATE_CONFIG_FILE`. It is applied on every `workflow_call` — both the
nightly sweep and any self-hosted caller workflow — before the repository's own
`.github/renovate.json5` is layered on top.

Currently it sets:

- **`gitIgnoredAuthors`** - the noreply email addresses of every GitHub App that
  may run Renovate against swept repositories (`odgbot`, `ocmbot`). Renovate
  treats commits by these authors as its own, so a branch written by one App's
  run is not skipped as "externally edited" when a different App's run picks it
  up later (e.g. central sweeper vs. a repo-local workflow with different
  credentials).

  > **Non-mergeable:** a repository-level `gitIgnoredAuthors` *replaces* this
  > list entirely. If you add your own entries, repeat the central ones too.
  >
  > To derive an App's noreply email:
  > ```bash
  > curl -s https://api.github.com/users/odgbot%5Bbot%5D | grep '"id"'
  > # -> <id>+<slug>[bot]@users.noreply.github.com
  > ```

Org-wide defaults (labels, timezone, schedule, automerge policies, etc.) will be
added here later and will apply to every repository without any per-repo configuration.

## How the Renovate setup works

- `renovate-schedule.yml` runs nightly and resolves the target list (or a
  `workflow_dispatch` override), then calls `renovate.yml` once per repository.
- `renovate.yml` first validates the central config (`renovate-config-validator`),
  then runs Renovate against the target repository.
- **Credentials:** write-capable runs use the `ODG_BOT` GitHub App
  (`vars.ODG_BOT_APP_ID` + `secrets.ODG_BOT_PRIVATE_KEY`), which is scoped to the
  **`renovate` environment** - jobs that need it declare that environment. Without
  App credentials (e.g. forks), runs degrade to a read-only dry-run
  (`RENOVATE_DRY_RUN=extract`); `dryRun: true` forces this even with credentials.
- **Commit authorship:** commits are made via the platform (`RENOVATE_PLATFORM_COMMIT`),
  so GitHub itself sets the App as author; no `gitAuthor` is configured centrally.
  PRs are opened under the App identity.
- **Config layering:** `.github/renovate-central.json5` is loaded on every
  `renovate.yml` call - nightly sweep and self-hosted workflows alike - and is
  the global config for every target. The repository's own `renovate.json[5]`
  still applies on top.
  Onboarding PRs are disabled (`RENOVATE_ONBOARDING: false`), so a repository must
  bring its own Renovate config before it is run.
- Renovate's repository cache lives in the caller repository's cache namespace,
  keyed per target.

## Onboarding a repository

There are two onboarding modes. Pick the one that fits your repository.

### Mode 1 - Nightly sweep (managed, simplest)

The central sweeper runs every repository on its list nightly. You hand off
scheduling and credentials entirely.

1. Add `.github/renovate.json5` to the repository (by convention, always use
   this path). Extend presets / add rules there; the central config at
   `.github/renovate-central.json5` in *this* repository applies on top.
   > **Note on non-mergeable arrays:** a repository-level `gitIgnoredAuthors`
   > *replaces* the central allowlist (the App identities whose branch commits
   > are accepted as renovate's own across sweeper/caller runs). Repeat the
   > central entries if you add your own.
2. Add the repository's `owner/name` to `.github/renovate-repositories.json`
   in this repository - the nightly sweep picks it up from the next run on.
3. Install the `ODG_BOT` GitHub App on the repository with the permissions
   requested in `renovate.yml` (contents, issues, pull-requests, statuses,
   workflows; read on checks/vulnerability-alerts).

### Mode 2 - Self-hosted workflow (own credentials, own schedule, central workflow)

Use this mode when you need your own schedule, triggers (push/PR), or want to
supply your own GitHub App / token instead of relying on `odgBot`.

Add a caller workflow to your repository that references the reusable workflow
here. The `odg-core` repository is a reference example:

```yaml
# .github/workflows/renovate.yml in your repository
name: Renovate
on:
  schedule:
    - cron: '0 1 * * *'       # nightly at 1 am UTC
  push:
    branches: [main]         # rebase/automerge after merges
  pull_request:
    branches: [main]
    paths:
      - .github/workflows/renovate.yml
      - .github/renovate.json5
  workflow_dispatch:
    inputs:
      repoCache:
        description: 'Repository cache: enabled, disabled, reset'
        type: choice
        default: enabled
        options: [enabled, disabled, reset]
      ignoreSchedule:
        type: boolean
        default: false
      logLevel:
        type: choice
        default: info
        options: [info, debug]
      useOdgbotCredentials:
        description: 'Use GitHub App credentials if configured (else: GITHUB_TOKEN)'
        type: boolean
        default: true

permissions:
  contents: read

jobs:
  renovate:
    uses: open-component-model/.github/.github/workflows/renovate.yml@v1
    secrets: inherit
    with:
      repository: ${{ github.repository }}
      repoCache: ${{ inputs.repoCache || 'enabled' }}
      ignoreSchedule: ${{ github.event_name == 'workflow_dispatch' && inputs.ignoreSchedule }}
      logLevel: ${{ inputs.logLevel || 'info' }}
      useOdgbotCredentials: ${{ github.event_name != 'workflow_dispatch' || inputs.useOdgbotCredentials }}
      dryRun: ${{ github.event_name == 'pull_request' }}
```

In this mode you do **not** need to be added to `.github/renovate-repositories.json`.
The workflow uses the credentials available in your repository's environment; if
`ODG_BOT` is installed and configured (`vars.ODG_BOT_APP_ID` +
`secrets.ODG_BOT_PRIVATE_KEY`), it is used for write runs; otherwise the job
falls back to a read-only dry-run. Your repository still needs its own
`.github/renovate.json5` (by convention, always use this path; same note about `gitIgnoredAuthors` applies).

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

1. Every push to `main` runs `release-please.yml`, which maintains an accumulating
   **release PR** (`chore(main): release X.Y.Z`) with changelog and version bump given that the commit-title had a `fix:` or `feat:` prefix. 
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
falls back to `GITHUB_TOKEN`, which requires that setting to be enabled.

## Consuming the shared workflows

Reusable workflows are pinned by git ref, like actions:

```yaml
uses: open-component-model/.github/.github/workflows/renovate.yml@v1     # floating major (recommended)
uses: open-component-model/.github/.github/workflows/renovate.yml@v1.1.0 # exact release
```

- `@v1.X.Y` never moves - fully reproducible.
- `@v1` automatically follows the latest compatible release; a breaking workflow
  change ships as `@v2` and does not affect `@v1` consumers.

The central config consumed at runtime comes from the same repository, on the
same major line as the workflow pin (`centralConfig` input, default
`open-component-model/.github@v1`). No caller configuration is needed in the
common case; forks and development setups override it once via the repository
variable `RENOVATE_CENTRAL_CONFIG` (`owner/repo@ref`) or per run via the input -
a SHA-pinning caller can pass the same commit for bit-exact reproducibility.

To verify that a floating tag tracks its release:

```bash
git ls-remote --tags https://github.com/open-component-model/.github 'v1*'
# refs/tags/v1 and refs/tags/v1.X.Y must show the same commit SHA
```
