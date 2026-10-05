# Central Renovate & Releases

How the centralized Renovate setup works, how repositories onboard, how the shared
workflows are versioned and released, and how consumers pin them.

## Components

| File | Purpose |
| ---- | ------- |
| `.github/renovate-central.json5` | Central/global Renovate configuration applied to every repository the sweeper runs (repository-local configs still apply on top) |
| `.github/renovate-repositories.yaml` | Commented YAML list of repositories (`owner/name`) covered by the hourly sweep; first entry doubles as the PR canary |
| `.github/workflows/renovate.yml` | Internal building block (`workflow_call`) of the sweeper: validates the central config and runs Renovate for ONE target repository. Called by `renovate-schedule.yml` (sweep matrix + PR canary) and via `workflow_dispatch` for single-repo debug runs - not an API for other repositories |
| `.github/workflows/renovate-validate.yml` | Thin reusable validator (`workflow_call`): `renovate-config-validator` on one repository file - seconds, read-only, no credentials. The one consumer-facing workflow |
| `.github/workflows/renovate-schedule.yml` | Hourly sweeper: fans the repository list out over the reusable workflow; on PRs it validates the config/list plus a dry-run canary |
| `.github/workflows/release-please.yml` | Release automation: release PR from conventional commits → tag `vX.Y.Z` + release → re-pin floating major tag `vX` |
| `release-please-config.json`, `.release-please-manifest.json` | release-please configuration and the version baseline |

## Central Renovate config (`.github/renovate-central.json5`)

This file is checked out at runtime by `renovate.yml` (via the `centralConfig`
input) and passed to Renovate as `RENOVATE_CONFIG_FILE`, layered UNDER each
repository's own `.github/renovate.json5`.

The sweeper does not pin a version of it: `renovate-schedule.yml` always
passes `github.repository@github.ref_name`, so the sweep consumes the config
at the very ref the run started from and config changes go live fleet-wide
on merge. The gate is the PR canary + validator, not a release. The `@v1`
default on the input only matters when nothing overrides it - forks and
local development (see Versioning and releases).

Currently it sets:

- **the shared base stack** - extends (recommended/best-practices, digest
  pinning, dependency dashboard), `minimumReleaseAge`, PR limits, automerge
  policy (off by default, on for patches and github-actions), major-update
  dashboard approval, vulnerability-alert overrides. Previously duplicated
  in consumer configs; now lives only here.
- **`gitIgnoredAuthors`** - the noreply emails whose commits on renovate
  branches count as Renovate's own (not "externally edited"): the GitHub Apps
  that run Renovate (`odgbot`, `ocmbot`) plus `github-actions[bot]`, so
  repository-local jobs running under GITHUB_TOKEN (e.g. post-processing
  pushes to `renovate/` branches) don't make renovate skip the branch.

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

- `renovate-schedule.yml` runs hourly and resolves the target list (or a
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
  `renovate.yml` call (sweep, PR canary, debug dispatch) and is the global
  config for every target. The repository's own `renovate.json[5]`
  still applies on top.
  Onboarding PRs are disabled and repository configs are optional: a listed
  repository without its own config runs on the central config alone
  (`RENOVATE_ONBOARDING: false` + `RENOVATE_REQUIRE_CONFIG: 'optional'`).
- Renovate's repository cache lives in the caller repository's cache namespace,
  keyed per target.

## Onboarding a repository

There is exactly one onboarding mode: the hourly sweep. (Push-reactive
dispatch - firing a run from the onboarded repository after merges - was
evaluated and dropped: an edge repository's GITHUB_TOKEN cannot call
`workflow_dispatch` on this repository, so no zero-credential push-reactive
variant exists. On-demand re-runs happen via `workflow_dispatch` on
`renovate.yml` / `renovate-schedule.yml` in this repository.)

### Hourly sweep (managed, simplest)

The central sweeper runs every repository on its list hourly. You hand off
scheduling and credentials entirely.

1. (Optional) Add `.github/renovate.json5` to the repository (by convention,
   always use this path) when you need repository-specific rules; the central
   config at `.github/renovate-central.json5` in *this* repository applies
   underneath. A repository without its own config still runs - on the
   central config alone.
   > **Note on non-mergeable arrays:** a repository-level `gitIgnoredAuthors`
   > *replaces* the central allowlist (the App identities whose branch commits
   > are accepted as renovate's own across sweeper/caller runs). Repeat the
   > central entries if you add your own.
2. Add the repository's `owner/name` to `.github/renovate-repositories.yaml`
   in this repository (YAML - comment freely) - the hourly sweep picks it up
   from the next run on.
3. Install the `ODG_BOT` GitHub App on the repository with the permissions
   requested in `renovate.yml` (contents, issues, pull-requests, statuses,
   workflows; read on checks/vulnerability-alerts).
4. (Optional) Validate config edits on the repository's own PRs - see
   [Validating a repository's renovate config on PRs](#validating-a-repositorys-renovate-config-on-prs)
   below.

## Validating a repository's renovate config on PRs

Edge repositories can validate their own `.github/renovate.json5` in pull
requests with the thin reusable validator - seconds, read-only, no
credentials:

```yaml
# .github/workflows/renovate-validate.yml in your repository
name: Validate renovate config
on:
  pull_request:
    paths:
      - .github/renovate.json5

permissions:
  contents: read

jobs:
  validate:
    uses: open-component-model/.github/.github/workflows/renovate-validate.yml@v1
    # with:
    #   configFile: .github/renovate.json5  # default; point elsewhere if needed
```

It runs `renovate-config-validator` with the same pinned renovate version as
the sweeper (one renovate PR bumps all pins together) and answers "is the
file valid". Whether it *behaves* as expected is answered by a dry-run of
`renovate.yml` in this repository (`workflow_dispatch` with `dryRun: true`) -
heavier (minutes), run on demand.

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
uses: open-component-model/.github/.github/workflows/renovate-validate.yml@v1     # floating major (recommended)
uses: open-component-model/.github/.github/workflows/renovate-validate.yml@v1.1.0 # exact release
```

- `@v1.X.Y` never moves - fully reproducible.
- `@v1` automatically follows the latest compatible release; a breaking workflow
  change ships as `@v2` and does not affect `@v1` consumers.

`renovate.yml`'s `centralConfig` input resolves to
`vars.RENOVATE_CENTRAL_CONFIG` and falls back to
`open-component-model/.github@v1` - i.e. the config on the same major line
as a workflow pinned to `@vX`. The sweeper overrides it anyway (same ref as
the run), so the fallback only matters for forks/development setups and
ad-hoc `workflow_dispatch` runs. A caller wanting bit-exact reproducibility
can pass `owner/repo@<same-sha>` explicitly.

To verify that a floating tag tracks its release:

```bash
git ls-remote --tags https://github.com/open-component-model/.github 'v1*'
# refs/tags/v1 and refs/tags/v1.X.Y must show the same commit SHA
```
