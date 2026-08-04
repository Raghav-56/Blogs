# 13. deploy scripts and CI

Your deploy is currently: SSH in, `cd` somewhere, `git pull`, run a build, copy files,
reload the proxy, restart the app, curl the site to check. Eight commands you half
remember, in an order you get wrong at 1am.

Then you forget the proxy reload and spend twenty minutes debugging a routing change that
was never applied.

## The principle: thin YAML, fat script

The temptation is to put those eight commands into a GitHub Actions workflow. Do not. Put
them in a script, in the repo, and have CI call the script.

This box's own docs state the reasoning:

> Runs on the **VPS** over SSH. Delegating the build and service reload steps to a script
> inside the repository keeps the CI yaml file simple and enables developers to run the
> exact same deployment commands locally.

That second clause is the real win. **You can run your deploy by hand, identically, when
CI is broken or you are in a hurry.** Logic that lives only in YAML can only be tested by
pushing, and debugging it means twelve commits called "fix ci".

## The script

A subcommand dispatcher, so each piece is independently runnable:

```bash
case "${1:-deploy}" in
    build)  cmd_build  ;;
    caddy)  cmd_caddy  ;;
    bun)    cmd_bun    ;;
    reload) BUILD_STEPS=""; cmd_caddy; cmd_bun; cmd_notify "reload" "success" ;;
    status) cmd_status ;;
    deploy) cmd_deploy ;;
    *) err "usage: deploy.sh [deploy|build|caddy|bun|reload|status]" ;;
esac
```

`./deploy.sh` does everything. `./deploy.sh caddy` just syncs routing. `./deploy.sh
status` shows you what is running. Fixing one thing does not mean re-running all of it.

Every script starts the same way:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_NAME="deploy"
ROOT="$(cd "$(dirname "$0")" && pwd)"
source "$ROOT/scripts/utils.sh"

require jq
require oxmgr
require caddy
```

**`set -euo pipefail` on line 2 of every shell script you write.** Exit on error, exit on
undefined variable, and let a failure anywhere in a pipeline fail the pipeline. Without
`-e`, a failed build is followed cheerfully by a deploy of the previous build's output.

`ROOT="$(cd "$(dirname "$0")" && pwd)"` makes the script work no matter what directory you
run it from.

`require` is twelve characters of insurance:

```bash
require() { command -v "$1" &>/dev/null || err "missing required tool: $1"; }
```

It converts "the deploy half-ran and then something strange happened at step four" into
`missing required tool: glow`, before anything is touched.

## Validate before you reload

```bash
cmd_caddy() {
    local src="$ROOT/raghav56.tech.caddy"
    if [[ ! -f "$src" ]]; then
        warn "no caddyfile at $src --- skipping"
        return
    fi
    log "syncing caddyfile..."
    sudo cp "$src" "$CADDY_DEST"
    sudo caddy validate --config /etc/caddy/Caddyfile
    sudo caddy reload  --config /etc/caddy/Caddyfile
    ok "caddy reloaded"
}
```

Because of `set -e`, a failed validation aborts the script before the reload runs. A typo
in your routing config becomes a failed deploy instead of an outage. The nginx form is
`nginx -t && systemctl reload nginx`.

Note where the config comes from: **the application repo**. The web server configuration
is versioned next to the code, reviewed like code, revertable like code, and a fresh
server can be rebuilt from it.

## Build ordering, learned the hard way

```bash
# Astro clears dist/ on build — run it first, then layer the bash-built assets on top
log "browser site  → astro build"
(cd "$ROOT" && bun run --silent build)

log "terminal card → dist/index.txt"
bash "$ROOT/scripts/terminal_card.sh" > "$DIST/index.txt"
```

That comment is the whole value of the line. The static site generator wipes its output
directory, so anything you generate has to come after it. Doing it the other way produces
a build that works locally when the directory already had the files, and mysteriously
loses them in CI where the directory starts empty.

Then:

```bash
if [[ "$WEBROOT" != "$DIST" ]]; then
    log "sync dist/ → $WEBROOT"
    rsync -a --delete "$DIST/" "$WEBROOT/"
else
    log "dev mode — dist/ is the webroot, skipping sync"
fi
```

`rsync -a --delete` mirrors, so pages you deleted from the source disappear from the live
site. Without `--delete` you accumulate stale routes forever, and an old page you thought
you removed stays online.

The `WEBROOT` check is how one script serves both dev (build into `dist/`, serve from
there) and prod (build, then sync). Environment symmetry: the dev server exercises the
same routing rules as production, so terminal-vs-browser behaviour is testable locally.

One step is deliberately non-fatal:

```bash
if ! bash "$ROOT/scripts/build_resume.sh" "$DIST"; then
    warn "failed to build resume assets, continuing main build"
fi
```

Decide, per step, whether a failure should stop the deploy. A missing optional asset
should not take the site down.

## The workflow

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
              exit 1
            fi
            git pull https://x-access-token:$GH_PAT@github.com/Raghav-56/raghav56.tech.git main
            git submodule sync && git submodule update --init
            chmod +x deploy.sh build.sh scripts/*.sh
            ./deploy.sh deploy

      - name: Health Check
        id: healthcheck
        run: bash .github/scripts/health-check.sh

      - name: Send Discord Notification
        if: always()
        env:
          DEPLOY_STATUS: ${{ job.status }}
          HEALTH_STATUS: ${{ steps.healthcheck.outputs.status }}
        run: bash .github/scripts/discord-notify.sh
```

`git push` is now the deploy command. Five things in that file are worth your attention.

**1. Actions pinned to commit SHAs.**

```yaml
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
```

A tag like `@v4` is a moving pointer the action's author can repoint at any time. If their
account is compromised, `@v4` silently becomes someone else's code running with your
secrets in scope, including your production SSH key. A SHA cannot be repointed. The
trailing version comment keeps it readable.

**2. `permissions: contents: read`.** Without this the automatic token gets whatever the
repository default is, often write. This job only needs to read.

**3. The PATH export, which is chapter 03 arriving on schedule.** `ssh host "command"` is
non-interactive: it reads no `.bashrc`, no `.bash_profile`. Everything in your home
directory is missing. The failure is `bun: command not found` in a CI log for a command
that works perfectly when you SSH in by hand, and it costs an afternoon the first time.

Note also `envs: GH_PAT`. Environment variables have to be explicitly forwarded into the
remote shell; they do not travel automatically.

**4. The dirty-working-tree guard.** This is scar tissue, and it is the most valuable
thing in the file:

```bash
if [ -n "$(git status --porcelain --untracked-files=no)" ]; then
  echo "ERROR: server working tree has local changes; refusing to pull." >&2
```

You SSH in to hotfix production. You edit a file. You forget. Someone pushes to `main`.
CI pulls, and either clobbers your fix or fails halfway through a merge conflict, leaving
the tree unbuildable **in the middle of a deploy**. The guard turns silent corruption into
a loud message that tells you exactly what to do.

The most recent commit in this repo is literally `ci: fail deploy when server git pull
fails; require health check`. It was learned, not designed.

**5. `if: always()` on the notification.** Without it, the notify step is skipped when the
deploy fails, which is precisely when you want to be told.

## Health checks that test the feature

```bash
ENDPOINTS=(
  "https://raghav56.tech|terminal-card|-A curl"
  "https://raghav56.tech|browser-page|-A Mozilla/5.0"
  "https://raghav56.tech/api/profile|api-profile"
  "https://raghav56.tech/health|bun-health"
)
```

The same URL is checked twice with different User-Agent headers, because this site serves
different content to terminals and browsers. Checking it once would prove half the system
works.

**A health check should exercise the feature, not just the port.** "The server responded"
is a much weaker claim than "the thing I built still does the thing".

The loop retries `MAX_RETRIES=10` times with `RETRY_DELAY=5`, so a check running
immediately after a reload does not fail on a race. And `curl --fail --silent --show-error
-L --max-time 10`: `--fail` makes a 500 a non-zero exit, `--max-time` stops a hung request
stalling the whole deploy.

## Two bash patterns worth keeping

**Returning several values from a child script.** A child process cannot modify its
parent's environment, so `export` in a script you call with `bash foo.sh` goes nowhere.
When you need multiple values and the script also prints human-readable output:

```bash
health_out="$(mktemp)"
set +e
HEALTH_OUTPUT_FILE="$health_out" bash "$ROOT/scripts/health-check.sh"
local hc_exit=$?
set -e
source "$health_out"
rm -f "$health_out"
```

The child writes `KEY=value` lines; the parent sources them. Note `set +e` around the call
so a failing health check does not abort the notification, with the exit code captured
manually.

**The CI shim.** All logic lives in `scripts/health-check.sh`, which knows nothing about
GitHub. A thin wrapper in `.github/scripts/` translates:

```bash
if [ -n "${GITHUB_OUTPUT:-}" ]; then
  echo "status=${HEALTH_STATUS:-failed}"  >> "$GITHUB_OUTPUT"
  echo "failed_urls=${FAILED_URLS:-}"     >> "$GITHUB_OUTPUT"
fi
```

The same pattern is in the notification script: every GitHub variable has a local git
fallback (`COMMIT_SHA` from `git rev-parse HEAD`, `REPO` from the origin URL, `ACTOR` from
`$USER`), so the identical script runs from CI or from your laptop. Portable logic, thin
platform adapter. If you move off GitHub Actions, you rewrite the shims and nothing else.

One more detail in the notify script worth copying: it builds its JSON payload with
`jq -n --arg` rather than interpolating variables into a JSON string by hand. Hand-built
JSON breaks the first time a commit message contains a quote character.

## Being honest about what this does not do

There is no build on the CI runner. The runner checks out the code only so the health and
notify scripts exist; the actual build happens on the server.

That is a deliberate simplification for a one-server personal site, and it has a real
consequence: **a broken build takes the site down**, because `rsync --delete` runs against
the live web root. A build-into-a-new-directory-then-swap-a-symlink deploy would make that
atomic and instantly revertable.

For this scale, the health check plus a Discord ping is an accepted tradeoff. Know that
you are making it.

## Dead config is not security

One repository secret on this project is referenced by nothing. The repo's own docs flag
it:

> **Configured but currently unused:** `DEV_BASIC_AUTH` is set as a repo secret but
> nothing in the workflow or scripts references it.

Auditing your own setup and writing down what you found is worth doing. A secret that
looks like it protects something and does not is worse than no secret, because you stop
looking.

Next: where the secrets actually live.
