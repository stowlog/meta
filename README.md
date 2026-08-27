# meta

Stowlog shared GitHub Actions workflows and configuration.

## Organization rulesets

Source of truth for org-level branch rulesets (apply via GitHub API / org settings — not auto-loaded from this path):

| File | Purpose |
|---|---|
| [`.github/rulesets/protect-key-branches.json`](./.github/rulesets/protect-key-branches.json) | Protect default, `production`, and `release/**` across all repos |

The second `bypass_actors` entry (`actor_type: "Team"`, `actor_id: 0`) is a placeholder for the
identity behind the `RELEASE_PLEASE_TOKEN` secret — needed so `vercel-release-deploy.yml`'s
`release-please` job can auto-merge its own version-bump PR (single manual trigger does
release + deploy). Setup, once per org:

1. Create a machine/bot GitHub user (or GitHub App) dedicated to release automation.
2. Generate a fine-grained PAT for it (repo contents + pull-requests: write) and store it as
   the `RELEASE_PLEASE_TOKEN` org secret.
3. Add that account to a team (e.g. `release-bots`).
4. In the GitHub ruleset editor's Bypass list, search for that team and add it — this fills in
   the real numeric `actor_id`, replacing the `0` placeholder here.
5. `bypass_mode: "pull_request"` means the bypass only applies when merging via a pull request
   (as `gh pr merge` does) — never on a direct push to the branch.

Bypassing this ruleset skips the *review* requirement only in practice, because the workflow
itself still waits on `lint`/`test`/`build` (`gh pr checks --watch --fail-fast`) before merging.

## Reusable Workflows

### `release-please-standard.yml` — Independent repos

Tracks conventional commits, opens a Release PR, and creates a GitHub Release on merge.

**Usage** — copy to `.github/workflows/release.yml` in your repo:

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-standard.yml@main
    secrets: inherit
    # with:
    #   target-branch: main          # default
    #   release-type: node           # default
    #   changelog-types: ''          # default (empty)
```

> [!NOTE]
> `RELEASE_PLEASE_TOKEN` is typically defined at the organization level. It must be explicitly declared in the reusable workflow to be accessible via `workflow_call`.

| Secret | Description |
|---|---|
| `RELEASE_PLEASE_TOKEN` | GitHub token with write permissions (falls back to GITHUB_TOKEN) |

---

### `release-please-monorepo.yml` — Monorepos

Manifest-based releases across multiple packages. Optionally publishes to npm.

**Requires in repo root:**
- `release-please-config.json`
- `.release-please-manifest.json`

**Usage — GitHub Releases only** (e.g. `stowlog/kiosks`):

```yaml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
```

**Usage — with npm publish** (e.g. `stowlog/foundations`):

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  release:
    uses: stowlog/meta/.github/workflows/release-please-monorepo.yml@main
    secrets: inherit
    with:
      publish-npm: true
      package-manager: pnpm
      build-command: 'pnpm build'
```

> Requires the `npm` GitHub environment configured with npm OIDC (Trusted Publishers).

---

### `vercel-deploy.yml` — Single deploy

Pulls project settings + env vars from Vercel, builds, and deploys one environment (`preview` or `production`).

**Usage** — copy to `.github/workflows/deploy.yml` in your repo:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  preview:
    if: github.event_name == 'push'
    uses: stowlog/meta/.github/workflows/vercel-deploy.yml@main
    secrets: inherit
    with:
      environment: preview

  production:
    if: github.event_name == 'workflow_dispatch'
    uses: stowlog/meta/.github/workflows/vercel-release-deploy.yml@main
    secrets: inherit
```

### `vercel-release-deploy.yml` — Release-gated production deploy

Runs release-please, resolves the tag to deploy, and calls `vercel-deploy.yml` with `environment: production`. See [`docs/examples/`](./docs/examples/) for full callers, including a monorepo variant.

> Manual-release flow: merge the release-please Release PR when ready, then run this workflow (`workflow_dispatch`) to promote it to production. Running it before merging just refreshes the Release PR — there's nothing to deploy yet.

#### Vercel project setup

- **Git integration off.** Disconnect Git in Vercel Project Settings, or set the Ignored Build Step to `exit 0` — otherwise every push also triggers an uncontrolled Vercel deployment that races this workflow.
- **Secrets.** Org secrets `VERCEL_TOKEN`, `VERCEL_ORG_ID`; repo secret `VERCEL_PROJECT_ID`.
- **Env vars live in Vercel, not GitHub.** These workflows use `vercel pull` to fetch build-time env vars and project settings straight from the Vercel project per environment, so `vercel build` matches what Vercel itself would build — a var that only exists in GitHub secrets will be missing at build time.

---

## Reference

### `release-please-standard.yml`

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track |
| `release-type` | string | `node` | release-please type |
| `changelog-types` | string | `''` | JSON array of extra commit types |

#### Secrets

| Secret | Description |
|---|---|
| `RELEASE_PLEASE_TOKEN` | GitHub token with write permissions (falls back to GITHUB_TOKEN) |

### `release-please-monorepo.yml`

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `target-branch` | string | `main` | Branch to track |
| `publish-npm` | boolean | `false` | Publish packages to npm |
| `node-version` | string | `20` | Node.js version for publish job |
| `package-manager` | string | `npm` | `npm` or `pnpm` |
| `build-command` | string | `''` | Command to run before publishing (e.g. `pnpm build`) |

#### Secrets

| Secret | Description |
|---|---|
| `RELEASE_PLEASE_TOKEN` | GitHub token with write permissions (falls back to GITHUB_TOKEN) |

### `vercel-deploy.yml`

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `environment` | string | *(required)* | `preview` or `production` |
| `vercel-target` | string | `''` | Custom Vercel environment name (overrides `--target`) |
| `ref` | string | `''` | Git ref to check out and deploy; empty uses the caller's ref |
| `working-directory` | string | `.` | Directory containing the Vercel project (monorepo app path) |
| `node-version` | string | `24` | Node.js version |
| `install-command` | string | `pnpm install --frozen-lockfile` | Skipped if empty |
| `vercel-cli-version` | string | `latest` | Vercel CLI version to install |
| `deploy-args` | string | `''` | Extra args appended to `vercel deploy` |
| `extra-env` | string | `''` | Newline-separated `KEY=VALUE` pairs exported before build (non-secret only) |
| `github-environment` | string | `''` | GitHub environment for the deploy job; defaults to `environment` |

#### Secrets

| Secret | Description |
|---|---|
| `VERCEL_TOKEN` | Vercel access token (required) |
| `VERCEL_ORG_ID` | Vercel organization (team) ID |
| `VERCEL_PROJECT_ID` | Vercel project ID |

#### Outputs

| Output | Description |
|---|---|
| `deployment-url` | URL of the created deployment |

### `vercel-release-deploy.yml`

#### Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `release-mode` | string | `latest-tag` | `latest-tag` (deploy newest GitHub Release) or `head` (deploy target-branch HEAD) |
| `target-branch` | string | `main` | Branch release-please tracks |
| `release-type` | string | `node` | release-please release type |
| `ref` | string | `''` | Explicit ref/tag to deploy (rollback); overrides `release-mode` resolution |
| `working-directory` | string | `.` | Directory containing the Vercel project |
| `node-version` | string | `24` | Node.js version |
| `install-command` | string | `pnpm install --frozen-lockfile` | Skipped if empty |
| `deploy-args` | string | `''` | Extra args appended to `vercel deploy` |
| `github-environment` | string | `production` | GitHub environment for the production deploy job |

#### Secrets

| Secret | Description |
|---|---|
| `VERCEL_TOKEN` | Vercel access token (required) |
| `VERCEL_ORG_ID` | Vercel organization (team) ID |
| `VERCEL_PROJECT_ID` | Vercel project ID |
| `RELEASE_PLEASE_TOKEN` | GitHub token with write permissions (falls back to GITHUB_TOKEN) |

#### Outputs

| Output | Description |
|---|---|
| `deployment-url` | URL of the created production deployment |
| `deployed-ref` | Ref/tag that was deployed |

---

## Examples

See [`docs/examples/`](./docs/examples/) for ready-to-copy caller workflows.
