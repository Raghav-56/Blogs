# Research: the deploy scripts

All at `/home/ubuntu/raghav/raghav56.tech/`. Repo `Raghav-56/raghav56.tech`, branch
`main`. Astro site plus a Bun API, with a parallel terminal rendering of every page.

The governing principle, quoted from `docs/deploy-pipeline.md`:

> Runs on the **VPS** over SSH. Delegating the build and service reload steps to a script
> inside the repository keeps the CI yaml file simple and enables developers to run the
> exact same deployment commands locally.

## scripts/utils.sh, sourced by everything

```bash
# CLI formatting helper functions & colors

RESET=$'\e[0m'
BOLD=$'\e[1m'
DIM=$'\e[2m'
CYAN=$'\e[1;36m'
GREEN=$'\e[1;32m'
MAGENTA=$'\e[1;35m'
YELLOW=$'\e[1;33m'
RED=$'\e[1;31m'
BLUE=$'\e[94m'

log()  { printf "${CYAN}[%s]${RESET} %s\n" "${SCRIPT_NAME:-sys}" "$*"; }
ok()   { printf "${GREEN}[ ✓ ]${RESET} %s\n" "$*"; }
warn() { printf "${YELLOW}[ ! ]${RESET} %s\n" "$*"; }
err()  { printf "${RED}[err]${RESET} %s\n" "$*" >&2; exit 1; }

require() { command -v "$1" &>/dev/null || err "missing required tool: $1"; }
```

Twelve lines that every one of these scripts uses. `require` is the important one: it
turns "the deploy half-ran and then something weird happened at step 4" into "missing
required tool: glow" before anything is touched. `err` writes to stderr and exits.

`$'\e[1;36m'` is bash ANSI-C quoting, which is how you get a literal escape byte into a
variable without `echo -e`.

## build.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_NAME="build"
ROOT="$(cd "$(dirname "$0")" && pwd)"
source "$ROOT/scripts/utils.sh"

DIST="$ROOT/dist"
WEBROOT="${WEBROOT:-/var/www/raghav56.tech}"

require jq
require rsync
require glow
require bun

# Astro clears dist/ on build — run it first, then layer the bash-built assets on top
log "browser site  → astro build"
(cd "$ROOT" && bun run --silent build)

log "terminal card → dist/index.txt"
bash "$ROOT/scripts/terminal_card.sh" > "$DIST/index.txt"

log "terminal pages → dist/*/index.txt"
bash "$ROOT/scripts/render_terminal.sh" "$DIST"

log "resume assets → dist/resume.{pdf,txt}"
if ! bash "$ROOT/scripts/build_resume.sh" "$DIST"; then
    warn "failed to build resume assets, continuing main build"
fi

if [[ "$WEBROOT" != "$DIST" ]]; then
    log "sync dist/ → $WEBROOT"
    rsync -a --delete "$DIST/" "$WEBROOT/"
else
    log "dev mode — dist/ is the webroot, skipping sync"
fi

ok "build complete"
```

Points worth teaching:

- `set -euo pipefail` on line 2 of every script. Exit on error, exit on undefined
  variable, and make a failing command in a pipeline fail the pipeline.
- `ROOT="$(cd "$(dirname "$0")" && pwd)"` makes the script work regardless of the
  directory you invoke it from.
- The ordering comment is the kind of thing only experience produces: Astro wipes `dist/`,
  so the bash-generated files have to be layered *after* it, not before.
- One step is deliberately non-fatal. A missing resume submodule warns and continues
  rather than failing the whole deploy.
- `WEBROOT="${WEBROOT:-/var/www/...}"` plus the equality check is how one script serves
  both dev (build into `dist/`, serve from there) and prod (build then rsync).
- `rsync -a --delete` mirrors, so files deleted from the source disappear from the web
  root. Without `--delete` you accumulate stale pages forever.

## deploy.sh

Subcommand dispatcher. The whole tail of the file:

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

The proxy-config sync, which is the pattern most worth stealing:

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

**Validate before reload.** With `set -e`, a failed `caddy validate` aborts the script
before `reload` runs, so a typo can never take the site down. The nginx equivalent is
`nginx -t && systemctl reload nginx`.

The web server configuration lives **in the application repo** and is copied to
`/etc/caddy/sites-enabled/` on deploy. Routing changes get code review and history like
everything else, and a fresh server can be reproduced from the repo.

And the multiple-return-values idiom, worth its own box in the chapter:

```bash
cmd_notify() {
    local event_type="${1:-deploy.sh}"
    local deploy_status="${2:-success}"
    # Run health check in a child shell — it can't export HEALTH_STATUS /
    # FAILED_URLS back to us, so it writes them to a file we source instead.
    local health_out
    health_out="$(mktemp)"
    set +e
    HEALTH_OUTPUT_FILE="$health_out" bash "$ROOT/scripts/health-check.sh"
    local hc_exit=$?
    set -e
    # shellcheck disable=SC1090
    source "$health_out"
    rm -f "$health_out"
    ...
}
```

A child process cannot modify its parent's environment. `export` in a script you invoke
with `bash foo.sh` goes nowhere. Options are: print one value to stdout and capture it,
`source` the script into the current shell, or (when you need several values and the
script also prints human output) write `KEY=value` to a temp file and `source` it. Note
`set +e` around the call so a failing health check does not abort the notification, with
the exit code captured manually.

## dev.sh

```bash
cmd_start() {
    cmd_build
    local port="${PORT:-}"
    if [[ -z "$port" && -f "$ROOT/.env" ]]; then
        port=$(grep -E "^PORT=" "$ROOT/.env" | cut -d= -f2 || true)
    fi
    port="${port:-3056}"
    # Kill any orphan process already holding this port
    local held_pid
    held_pid=$(ss -tlnp "sport = :$port" 2>/dev/null | grep -oP 'pid=\K[0-9]+' | head -1 || true)
    if [[ -n "$held_pid" ]]; then
        warn "port $port held by PID $held_pid - killing it"
        kill "$held_pid" 2>/dev/null || true
        sleep 0.5
    fi
    log "browser  → http://localhost:$port"
    log "terminal → curl -A curl http://localhost:$port"
    PORT="$port" exec bun run --hot "$ROOT/server.js"
}
```

`EADDRINUSE` from a previous run you forgot about is the most common local-dev
annoyance. Three lines fix it forever:

```bash
ss -tlnp "sport = :$port" | grep -oP 'pid=\K[0-9]+'
```

`grep -oP 'pid=\K[0-9]+'` is a PCRE trick: `\K` discards everything matched so far, so
only the digits are printed. Also note `exec` on the final line, which replaces the shell
with bun so Ctrl-C and signals go to the right process instead of to a wrapper.

## scripts/health-check.sh

The header comment is the documentation:

```
# Outputs (for callers — this script always runs as a child process, so
# `export` alone never reaches the caller's shell):
#   HEALTH_STATUS   – "passed" or "failed"
#   FAILED_URLS     – comma-separated list of failed URLs (empty if passed)
# If HEALTH_OUTPUT_FILE is set, these are also written there as KEY=value
# lines the caller can `source` after this script exits.
```

The endpoint table:

```bash
ENDPOINTS=(
  "https://raghav56.tech|terminal-card|-A curl"
  "https://raghav56.tech|browser-page|-A Mozilla/5.0"
  "https://raghav56.tech/api/profile|api-profile"
  "https://raghav56.tech/health|bun-health"
)
```

The same URL is checked twice with different User-Agent headers. Because this site serves
different content to terminals and browsers, checking it once would only prove half the
system works. **A health check should exercise the feature, not just the port.**

Retry loop with `MAX_RETRIES=10` and `RETRY_DELAY=5`, so a check run immediately after a
reload does not fail on a race. `curl --fail --silent --show-error -L --max-time 10`:
`--fail` makes a 500 a non-zero exit, `--max-time` prevents a hang from stalling the
deploy.

## The other build scripts, in brief

- **`scripts/load_profile.sh`** reads `content/profile.json` and uses
  `jq -r ... @sh` to emit safely-quoted `key='value'` lines, then `eval`s them. One `jq`
  process for all values instead of one per key. Single source of truth for both the
  HTML and terminal renderings.
- **`scripts/terminal_card.sh`** draws the `curl raghav56.tech` box with Unicode box
  characters, 54 columns wide. Contains a pure-bash ANSI stripper that needs the
  `extglob` set in `.bashrc`:

  ```bash
  strip_ansi() { printf '%s' "${1//$'\e'\[+([0-9;])m/}"; }
  ```

  Padding is computed on the *visible* length after stripping, not the byte length,
  which is why a coloured line still lines up with the box edge.
- **`scripts/render_terminal.sh`** pipes markdown through
  `glow - -s config/glow_style.json -w 100` with `CLICOLOR_FORCE=1` (glow disables colour
  when stdout is not a TTY; that variable forces it back on). Frontmatter is parsed by
  three small awk functions rather than a YAML dependency. Sorting by date is done by
  emitting `date\tpath` and piping through `sort -r | cut -f2`, the classic decorate,
  sort, undecorate.
- **`scripts/build_resume.sh`** pulls from the `resume/` git submodule and preprocesses
  the markdown with three `sed` passes: strip `[[wikilinks]]`, flatten `[label](url)` to
  `label`, and escape `@` as `&#64;` so glow does not autolink the email address.
- **`scripts/notify.sh`** builds a Discord embed with `jq -n --arg` / `--argjson` and
  POSTs it. It never interpolates values into a JSON string by hand, which is the
  injection bug you get for free otherwise. Every GitHub Actions variable has a local git
  fallback (`COMMIT_SHA` from `git rev-parse HEAD`, `REPO` from the origin URL, `ACTOR`
  from `$USER`), so the identical script runs from CI or from a manual `./deploy.sh`.
  Service state is scraped from `oxmgr list` and `systemctl is-active caddy`. It no-ops
  silently when `DISCORD_WEBHOOK` is unset.
