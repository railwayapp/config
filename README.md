# Railway Config Action

Plan [Railway Infrastructure as Code](https://docs.railway.com/infrastructure-as-code) on pull requests and apply the reviewed plan on merge.

The plan job pins three things into an artifact: the change set, the environment's `configEtag`, and the `.railway/` git tree. The apply job applies **that artifact** — it never re-evaluates the authoring file. If the environment changed since the plan, or the merged `.railway/` tree is not the planned tree, the apply fails and you re-plan. What you reviewed is what lands.

## Usage

Create a [project token](https://docs.railway.com/guides/public-api#creating-a-token) for the target environment and store it as the `RAILWAY_TOKEN` repository secret. Then:

```yaml
name: Railway config

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    paths:
      - ".railway/**"

permissions:
  contents: read
  pull-requests: write
  actions: read

jobs:
  plan:
    if: github.event.action != 'closed' && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: railwayapp/config@v1
        with:
          command: plan
          railway-token: ${{ secrets.RAILWAY_TOKEN }}

  apply:
    if: github.event.action == 'closed' && github.event.pull_request.merged && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.merge_commit_sha }}
      - uses: railwayapp/config@v1
        with:
          command: apply
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
```

Every PR touching `.railway/` gets a plan comment with the diff, marked when destructive. Merging is the approval: the apply job downloads the plan pinned to the PR's head commit and applies exactly that change set.

### Split workflows

The single-workflow layout above runs apply on the pull request's `closed` event, so the apply run doesn't show up on the merge commit itself. If you'd rather see a separately named apply run on your default branch, split it into two workflows — the action resolves the merged PR (and its pinned plan) from the pushed commit:

```yaml
# .github/workflows/railway-plan.yml
name: Railway plan

on:
  pull_request:
    paths: [".railway/**"]

permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: railwayapp/config@v1
        with:
          command: plan
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
```

```yaml
# .github/workflows/railway-apply.yml
name: Railway apply

on:
  push:
    branches: [main]
    paths: [".railway/**"]

# pull-requests: read lets the action resolve which merged PR the
# pushed commit came from, to fetch that PR's pinned plan artifact.
permissions:
  contents: read
  actions: read
  pull-requests: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: railwayapp/config@v1
        with:
          command: apply
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
```

Direct pushes to `main` that never went through a PR have no pinned plan, so the apply fails by design — every change to `.railway/` goes through a reviewed plan.

### Comment identity

With no extra setup, the plan comment is posted by whoever owns `github-token` — the default workflow token shows as **github-actions**.

To post under the official **Railway** bot (the same identity as PR-environment deploy comments), grant the plan job `id-token: write` and have the [Railway GitHub App](https://github.com/apps/railway-app) installed on the repo:

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write
```

The action sends a GitHub Actions OIDC token to Railway, which verifies that the workflow runs in this repo and posts the comment through the Railway GitHub App. No secrets beyond the `railway-token` you already provide. If the OIDC token or app installation is missing, the action falls back to posting with `github-token`.

Alternatively, to post under your own bot, mint a GitHub App installation token and pass it as `github-token`:

```yaml
      - uses: actions/create-github-app-token@v1
        id: app
        with:
          app-id: ${{ vars.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_PRIVATE_KEY }}
      - uses: railwayapp/config@v1
        with:
          command: plan
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
          github-token: ${{ steps.app.outputs.token }}
```

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `command` | — | `plan` (on PR) or `apply` (on merged PR, or push of a merged PR's commit) |
| `railway-token` | — | Railway project token for the target environment |
| `cli-version` | `latest` | `@railway/cli` version to install |
| `artifact-name` | `railway-plan` | Name of the plan artifact |
| `plan-path` | `railway-plan.json` | Path of the pinned plan file |
| `comment` | `true` | Post/update the PR plan comment (plan only) |
| `package-manager` | `auto` | Package manager for installing repository dependencies in plan mode: `auto`, `npm`, `pnpm`, `yarn`, `bun` or `none` |
| `confirm-destructive` | `true` | Allow destructive applies; the plan comment marks them, merge is the approval. Set `false` to make destructive applies fail |
| `github-token` | `github.token` | Token for artifacts and the PR comment |

## Failure modes, by design

- **Environment drifted** (dashboard edit, concurrent apply, another pipeline): apply fails with the etag mismatch. Re-open or re-push the PR to re-plan.
- **`.railway/` changed between plan and merge** (base moved, conflict resolution): apply fails on the tree check. Re-plan on the merged revision.
- **No plan artifact for the merged head SHA**: apply fails; wait for the plan job or re-open the PR.
- **Destructive changes with `confirm-destructive: false`**: apply fails instead of removing resources.

An empty change set applies as a no-op.

## Notes

- Requires the workflow permissions shown above: `pull-requests: write` for the comment, `actions: read` for the artifact download.
- Fork PRs don't get secrets; the `head.repo.full_name == github.repository` guard skips them.
- The action installs Node 24 (the TypeScript authoring file is evaluated with native type stripping) and installs your repository's dependencies in plan mode so `railway/iac` resolves.
- The package manager is detected, not assumed: `packageManager` in `package.json` first, then the lockfiles (`pnpm-lock.yaml`/`pnpm-workspace.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb`/`bun.lock` → bun, otherwise npm). pnpm and Yarn come from corepack, bun from `oven-sh/setup-bun`. Installs are lockfile-frozen when a lockfile is present. Override with `package-manager: pnpm` (or `none` to skip the install entirely, e.g. when a prior step already installed dependencies). The Railway CLI itself is always installed globally with npm, from outside the checkout, so a pnpm or Yarn workspace root can't interfere.
- Plan artifacts follow your repository's artifact retention (90 days by default) — far longer than any reasonable PR-merge gap.
