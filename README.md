# Journey Monitor runner

The GitHub Actions workflows that run Journey Monitor's journeys (SPEC §7.5, §7.7).

This folder (`infra/runner-repo` in the private app repo, `josephsilcock/journey-monitor`) is copied as the root of
the public repo `josephsilcock/journey-monitor-runner`, unchanged. The app repo's `sync-runner-repo.yml` workflow does
the copying: whenever the folder changes on the app repo's `main`, it mirrors the folder to the runner repo's `main`,
so the runner repo only ever holds what this folder holds. Don't edit the runner repo directly: the next sync
overwrites it. The workflows check out the app repo's `main` and run its worker, so changes to the worker itself need
no copy.

## Workflows

| Workflow | When | What it does |
|---|---|---|
| `run.yml` | Every 15 minutes, and on Run now (the web app dispatches it on `main`) | Claims queued runs, runs them in the Playwright image and uploads their media (`node apps/worker/src/cli.ts run`) |
| `maintenance.yml` | Daily at 03:00 UTC, and manually from the Actions tab | Retention and clean-up (`node apps/worker/src/cli.ts maintenance`) |

Both jobs run only in `josephsilcock/journey-monitor-runner` (a fork or another copy skips them), give the job's
own GitHub token no permissions, and pin every action to a commit SHA. Several `run.yml` jobs may run at once: claiming
is atomic.

The `run.yml` job uses the image `mcr.microsoft.com/playwright:v<version>-noble`, whose `<version>` must be the
`playwright` version in the app repo's `apps/worker/package.json`, so the browsers match. When Playwright is bumped,
bump the image tag here too (the sync copies it); the app repo's test fails until they match.

## Repository secrets

Set these under **Settings → Secrets and variables → Actions → Secrets** in the runner repo. All three are secrets,
not variables, so GitHub masks them in the public log.

| Secret | Value |
|---|---|
| `APP_REPO_TOKEN` | A fine-grained personal access token with `contents: read` on `josephsilcock/journey-monitor` only, used to check out the app. |
| `WORKER_TOKEN` | The web app's `WORKER_TOKEN` value. |
| `APP_BASE_URL` | The web app's `https` origin. It is a secret, not a variable, so the address is masked in the log. |

The web app's `GITHUB_DISPATCH_TOKEN` (set in its hosting, not here) is a fine-grained personal access token with
`actions: write` on `josephsilcock/journey-monitor-runner` only. It lets Run now dispatch `run.yml`.

The app repo's `RUNNER_REPO_TOKEN` (an Actions secret in the app repo, not here) is a fine-grained personal access
token with `contents: write` and `workflows: write` on `josephsilcock/journey-monitor-runner` only. The sync pushes
with it.

## Keeping the log clean

The repo is public, and so are its Actions logs. The workflows print only setup output for public packages and the
worker's logger lines (opaque run IDs, counts and timings), and upload no artifacts. To keep it that way:

- Never set `ACTIONS_STEP_DEBUG` or `ACTIONS_RUNNER_DEBUG` (as secrets or variables), and don't re-run jobs with debug
  logging.
- Don't allow pull requests from forks to run workflows (**Settings → Actions → General**). The workflows have no
  pull request triggers.
- Pass secrets to steps only through `env` or `with`, never as `${{ }}` inside a script.

## When a job fails

A failing journey never fails a job: its result is recorded in the app. A failed job means API or infrastructure
trouble, and its result is the worker's exit code:

- `64`: configuration, for example a missing or invalid secret;
- `69`: the web app's API failed or timed out, or a run was abandoned (for example reaped as "worker lost");
- `70`: an unexpected error.

The log shows only the kind of error. Runs a failed job left behind are marked `errored` by the app's reaper.

## Scheduled workflows stop after 60 days without activity

GitHub turns off scheduled workflows in a public repo after 60 days without repository activity, and emails the
owner. The runner repo only changes when this folder does, so this will happen in a quiet period. There is no
keep-alive workflow, by design.

While the workflows are off, no scheduled runs are created, and Run now can't start a job (its runs stay queued).

To fix it, open the runner repo's **Actions** tab, select each workflow (`run.yml` and `maintenance.yml`) and click
**Enable workflow**. A sync of a change to this folder also counts as activity.

GitHub's documentation: <https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/disabling-and-enabling-a-workflow>.
