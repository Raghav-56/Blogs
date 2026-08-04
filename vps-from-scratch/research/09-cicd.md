# Research: CI/CD

One workflow file, `.github/workflows/deploy.yml`. Push-based over SSH. No webhook
receiver, no bare repo with a `post-receive` hook, no polling agent. Verified: there are
no bare repos on the box, no `post-receive` hooks anywhere, and every listening port is
accounted for.

## The workflow, verbatim

```yaml
name: Deploy Portfolio

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
        with:
          fetch-depth: 2

      - name: Deploy over SSH
        uses: appleboy/ssh-action@2ead5e36573f08b82fbfce1504f1a4b05a647c6f # v1.2.2
        env:
          GH_PAT: ${{ secrets.GH_PAT }}
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          port: ${{ secrets.PORT }}
          envs: GH_PAT
          script: |
            set -euo pipefail
            export PATH="$HOME/.bun/bin:$HOME/.cargo/bin:$HOME/.local/bin:/usr/local/bin:/usr/bin:$PATH"
            cd ~/raghav/raghav56.tech
            if [ -n "$(git status --porcelain --untracked-files=no)" ]; then
              echo "ERROR: server working tree has local changes; refusing to pull." >&2
              git status --porcelain --untracked-files=no >&2
              echo "Commit, stash, or discard these changes on the server, then re-run the deploy." >&2
              exit 1
            fi
            git pull https://x-access-token:$GH_PAT@github.com/Raghav-56/raghav56.tech.git main
            git submodule sync && git submodule update --init
            chmod +x deploy.sh build.sh scripts/*.sh
            ./deploy.sh deploy

      - name: Health Check
        id: healthcheck
        env:
          MAX_RETRIES: 10
          RETRY_DELAY: 5
        run: bash .github/scripts/health-check.sh

      - name: Send Discord Notification
        if: always()
        env:
          DISCORD_WEBHOOK: ${{ secrets.DISCORD_WEBHOOK }}
          COMMIT_SHA: ${{ github.sha }}
          COMMIT_MSG: ${{ github.event.head_commit.message }}
          BRANCH: ${{ github.ref_name }}
          ACTOR: ${{ github.actor }}
          DEPLOY_STATUS: ${{ job.status }}
          HEALTH_STATUS: ${{ steps.healthcheck.outputs.status }}
          FAILED_URLS: ${{ steps.healthcheck.outputs.failed_urls }}
          REPO: ${{ github.repository }}
          RUN_ID: ${{ github.run_id }}
          WORKFLOW_TRIGGER: ${{ github.event_name }}
        run: bash .github/scripts/discord-notify.sh
```

## Six things in this file worth a paragraph each

**1. Actions pinned to commit SHAs, not tags.**

```yaml
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

A tag like `@v4` is a moving pointer that the action's author can repoint at any time. If
their account is compromised, `@v4` silently becomes someone else's code, running with
your secrets in scope. A SHA cannot be repointed. The trailing comment keeps it readable.

**2. Explicit least-privilege token.**

```yaml
permissions:
  contents: read
```

Without this the `GITHUB_TOKEN` gets the repository default, which is often write. This
workflow only needs to read code.

**3. Non-interactive SSH has no PATH.**

```bash
export PATH="$HOME/.bun/bin:$HOME/.cargo/bin:$HOME/.local/bin:/usr/local/bin:/usr/bin:$PATH"
```

`ssh host "command"` runs a non-interactive shell, which reads neither `.bashrc` nor
`.bash_profile`. Everything you installed to a home directory is missing. The failure is
`bun: command not found` in a CI log for a command that works perfectly when you SSH in
by hand, and it wastes an afternoon the first time.

**4. The dirty-working-tree guard.**

```bash
if [ -n "$(git status --porcelain --untracked-files=no)" ]; then
  echo "ERROR: server working tree has local changes; refusing to pull." >&2
  ...
  exit 1
fi
```

This is scar tissue. You SSH in to hotfix production, edit a file, forget. Someone pushes
to `main`, CI pulls, and your fix is either clobbered or the pull fails halfway with a
merge conflict, leaving the tree in an unbuildable state mid-deploy. The guard turns a
silent corruption into a loud, specific error message that tells you exactly what to do.

The most recent commit in the repo is literally
`ci: fail deploy when server git pull fails; require health check`. The guard was learned,
not designed.

**5. The pull uses a token even though the remote is SSH.**

```bash
git pull https://x-access-token:$GH_PAT@github.com/Raghav-56/raghav56.tech.git main
```

The server's checkout has an SSH remote, but CI passes an explicit HTTPS URL with a token
rather than relying on a server-side deploy key. Fewer moving parts on the server, and
the credential's lifetime is the job. Note `envs: GH_PAT` in the action config: variables
have to be explicitly forwarded into the remote shell, they do not travel automatically.

**6. `if: always()` on the notification.**

Without it, the notify step is skipped when the deploy fails, which is exactly when you
want to be told.

## The CI shim pattern

`.github/scripts/health-check.sh` is a thin wrapper. All the logic lives in
`scripts/health-check.sh`, which knows nothing about GitHub:

```bash
REPO_ROOT="$(git rev-parse --show-toplevel)"

HEALTH_OUTPUT_FILE="$(mktemp)"
trap 'rm -f "$HEALTH_OUTPUT_FILE"' EXIT

set +e
HEALTH_OUTPUT_FILE="$HEALTH_OUTPUT_FILE" bash "$REPO_ROOT/scripts/health-check.sh"
EXIT_CODE=$?
set -e

source "$HEALTH_OUTPUT_FILE"

# Write GHA step outputs
if [ -n "${GITHUB_OUTPUT:-}" ]; then
  echo "status=${HEALTH_STATUS:-failed}"     >> "$GITHUB_OUTPUT"
  echo "failed_urls=${FAILED_URLS:-}"        >> "$GITHUB_OUTPUT"
fi

exit $EXIT_CODE
```

The shim's only job is translating to the CI platform's conventions (appending
`key=value` to `$GITHUB_OUTPUT`). Everything real is portable and runnable on a laptop.
`trap ... EXIT` cleans up the temp file on every exit path including failure.

Same structure for `.github/scripts/discord-notify.sh`: it sets `EVENT_TYPE` and
`RUN_URL` from GitHub variables and delegates to `scripts/notify.sh`.

## Secrets used

`GH_PAT`, `HOST`, `USERNAME`, `PORT`, `SSH_KEY`, `DISCORD_WEBHOOK`. Note `PORT` here is
the SSH port, not an application port, which the repo docs flag explicitly because it is
confusing.

`DEV_BASIC_AUTH` exists as a repository secret and **nothing references it**. The repo's
own `docs/deploy-pipeline.md` calls this out and says to wire it up or delete it. Dead
secrets are worth a line in the chapter: they look like security and are not.

## What this pipeline does not do

No build on the runner. The runner checks out the code only so the health-check and
notify scripts are available; the actual build happens on the VPS. That is a deliberate
simplification for a single-server personal site, and it has one real consequence worth
stating honestly: **a broken build takes the site down**, because `rsync --delete` runs
against the live web root. A build-then-swap-symlink deploy would avoid that. For this
scale, the health check plus a Discord ping is the accepted tradeoff.
