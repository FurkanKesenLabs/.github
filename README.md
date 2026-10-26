# FurkanKesenLabs CI recipe

One reusable GitHub Actions workflow for the organization's Node projects: `.github/workflows/node-ci.yml`.

- **Runner is chosen by repository owner.** Inside `FurkanKesenLabs` jobs run on the label in the repository variable `CI_RUNNER` when it is set, otherwise on Blacksmith (`blacksmith-2vcpu-ubuntu-2404`); anywhere else (for example a repo still on a personal account) they run on GitHub-hosted `ubuntu-latest`. Nothing changes in a project's file when it is transferred into the organization.
- **Self-hosted Mac mini runners.** `CI_RUNNER=mini-linux` sends a repo's jobs to the one-job Linux arm64 runners on Mac Mini 16 (3 slots of 3 GB); `mini-linux-large` sends them to its 7 GB slot. They are Ubuntu 24.04 containers from GitHub's runner image, thrown away after each job, with no Docker socket and no access to the home network. `CI_RUNNER` has to be a repository variable, because GitHub Free hides organization variables from private repositories. Workflows that use deploy or backup secrets stay on hosted runners.
- **Two tiers.** `push` runs the fast contracts job (install + the commands you list, target under 2 minutes). `pull_request`, `schedule` and `workflow_dispatch` run the full suite (tests, build, end-to-end). A scheduled run skips itself when the branch head already has a green scheduled/dispatched run.
- **Fallback.** To move a project off the Mac mini, delete its variable (`gh variable delete CI_RUNNER -R FurkanKesenLabs/<name>`); jobs already queued for the mini must be cancelled and re-run. To move every project back to GitHub-hosted runners, change the label in `node-ci.yml` once. To move one run, pass `runner: ubuntu-latest`.

## Copy this into a project as `.github/workflows/ci.yml`

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "23 */6 * * *"   # full suite every 6 hours (skipped when main did not move)
  workflow_dispatch:

permissions:
  contents: read
  actions: read

concurrency:
  group: ci-${{ github.ref }}-${{ github.event_name == 'push' && 'fast' || 'full' }}
  cancel-in-progress: true

jobs:
  ci:
    uses: FurkanKesenLabs/.github/.github/workflows/node-ci.yml@main
    with:
      package-manager: pnpm            # or npm
      fast-commands: |
        pnpm typecheck
        pnpm build
      test-command: pnpm test
      build-command: pnpm build
```

## Inputs

| Input | Default | Meaning |
|---|---|---|
| `package-manager` | `pnpm` | `pnpm` or `npm`; selects the install command and the dependency cache |
| `node-version-file` | `.nvmrc` | Node version file, relative to `working-directory` |
| `working-directory` | `.` | Project root inside the repository |
| `fast-commands` | | Newline-separated commands for the fast tier, run after install |
| `test-command` | | Full tier test command (empty: skipped) |
| `build-command` | | Full tier build command (empty: skipped) |
| `e2e-command` | | Full tier end-to-end command (empty: skipped) |
| `env` | | Newline-separated `KEY=VALUE` lines exported to every command (for example `PW_CHANNEL=chrome`) |
| `cache-path` | | Extra directory to cache in the full tier (for example encoded assets) |
| `cache-name` | `extra` | Key prefix for that cache |
| `cache-key-files` | | Newline-separated git pathspecs whose content hashes into the cache key |
| `tier` | `auto` | `auto` (by event), `fast` or `full` |
| `runner` | | Force a `runs-on` label |
| `fast-timeout-minutes` | `10` | |
| `full-timeout-minutes` | `30` | |

Callers must grant `permissions: actions: read` for the scheduled-run guard; everything else needs only `contents: read`.

## Rules for agents (Claude Code, Codex)

1. New repositories start on the personal account. Throwaway projects stay there and get no CI.
2. The moment a project gets CI, it belongs in the organization: transfer first (`gh api repos/<owner>/<name>/transfer -f new_owner=FurkanKesenLabs`), fix the local remote, then add the `ci.yml` above. The runner follows the owner automatically.
3. At the start of work in a repository, if `origin` redirects to a new owner, update it.
4. Put CI logic changes here, not in individual projects; projects only pass commands.
5. Exception: a project deployed through Vercel on the Hobby plan stays on the personal account, because Vercel Hobby
   cannot deploy private repositories owned by an organization. Its `ci.yml` is the same; the recipe falls back to
   GitHub-hosted runners outside the organization.

Public on purpose: the recipe contains no secrets, and public reusable workflows can be called from private repositories under any owner.
