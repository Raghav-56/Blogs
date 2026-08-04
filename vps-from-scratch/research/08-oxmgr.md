# Research: oxmgr

**What it is:** a process manager written in Rust, in the same category as PM2. Not a
tool written for this box, and not widely known. Installed here with
`cargo install --git`, version 0.5.0, binary at `~/.cargo/bin/oxmgr` (4.1 MB, ARM
aarch64, stripped).

Upstream: `Vladimir-Urik/OxMgr` on GitHub. Source checkout lives in
`~/.cargo/git/checkouts/oxmgr-*/`.

## Live state

```
$ oxmgr list
+----+---------------+---------+--------+---------+---------+----------+------+---------+---------+
| ID | NAME          | STATUS  | MODE   | PID     | UPTIME  | RESTARTS | CPU% | RAM(MB) | HEALTH  |
+----+---------------+---------+--------+---------+---------+----------+------+---------+---------+
| 13 | raghav56-tech | running | single | 2426420 | 23d 21h | 0        | 0.0  | 53      | healthy |
+----+---------------+---------+--------+---------+---------+----------+------+---------+---------+
```

23 days, zero restarts, 53 MB resident. Real numbers beat adjectives.

## The config, from the committed template

`oxfile.toml.template` (the live `oxfile.toml` is gitignored because it holds a real API
key):

```toml
version = 1

[defaults]
restart_policy = "on_failure"
max_restarts = 5
restart_delay_secs = 5
stop_timeout_secs = 5

[[apps]]
name = "{{APP_NAME}}"
command = "{{BUN_PATH}}/bun run --hot server.js"
cwd = "{{PROJECT_ROOT}}"

health_cmd = "curl -fsS http://127.0.0.1:{{BUN_PORT}}/health"
health_interval_secs = 10
health_timeout_secs = 2
health_max_failures = 3

[apps.env]
NODE_ENV = "production"
PORT = "{{BUN_PORT}}"
HOSTNAME = "127.0.0.1"
# Contact form email delivery (https://resend.com API key).
# Without it, /api/contact logs the message and returns 503.
RESEND_API_KEY = "{{RESEND_API_KEY}}"
CONTACT_TO = "{{CONTACT_EMAIL}}"

[apps.logs]
stdout = "/var/log/{{APP_NAME}}/{{APP_NAME}}.log"
stderr = "/var/log/{{APP_NAME}}/{{APP_NAME}}-error.log"
```

The live file carries this comment above the env block, which is the single most useful
sentence in it:

```toml
# Env vars must be explicit — OxMgr daemon runs in a clean env,
# Bun does NOT auto-load .env when launched by a process manager.
# HOSTNAME is set explicitly to override the system hostname env var.
```

Three separate traps in three lines:

1. A supervisor daemon does not inherit your interactive shell environment. Whatever you
   set in `.bashrc` is invisible to it.
2. `bun run` auto-loads `.env` when you run it yourself, but the file lookup is relative
   to the working directory and the behaviour is not something to rely on under a
   supervisor. Declare the variables in the supervisor config.
3. `HOSTNAME` is already set by the system to the machine's hostname (`oracler`). If the
   app reads `process.env.HOSTNAME` to decide its bind address, it will try to bind
   `oracler` unless you override it. This is a genuinely nasty one, and it is why the
   override exists.

## What this buys over plain systemd

systemd restarts a process when it exits. It has no idea whether a process that is still
running is actually *serving*. A Node process that has deadlocked, or wedged its event
loop, or lost its database pool, stays "active (running)" forever.

```toml
health_cmd = "curl -fsS http://127.0.0.1:2056/health"
health_interval_secs = 10
health_timeout_secs = 2
health_max_failures = 3
```

Three consecutive failures, 30 seconds, and it restarts. That is an application-level
liveness probe, the same idea Kubernetes calls `livenessProbe`. You can approximate it in
systemd with `Type=notify` plus a watchdog, but the app has to cooperate.

The other wins: `[apps.logs]` routes stdout and stderr to named files per app without
touching journald config; a single `oxfile.toml` describes several apps; and the file
lives in the repo, so the service definition is version-controlled next to the code it
runs.

## Command surface

```
start  stop  restart(rs)  reload(rl)  pull  delete(rm)  list(ls|ps)  ui  logs(log)
status  import  export  apply  convert  validate  deploy  doctor  events  runtime
startup  service  daemon
```

It reads PM2 `ecosystem.config.{js,cjs,mjs,json}` files, and `oxmgr convert` migrates one
to the native `oxfile.toml`.

## Runtime layout

- `~/.local/share/oxmgr/state.json` (mode 0600) is the persisted process table.
- `~/.local/share/oxmgr/events.sock` is a unix socket for `oxmgr events`.
- `~/.local/share/oxmgr/logs/` holds rotated per-app logs. `raghav56-tech.out.log.1` is
  33 MB, which is a reminder to check rotation settings on anything chatty.

## The supervision stack

```
systemd (system)         →  caddy
systemd (user) + linger  →  oxmgr daemon  →  bun run --hot server.js
```

Two layers, deliberately. systemd survives reboot and starts the oxmgr daemon; oxmgr does
per-app health checking and log routing. The systemd user unit:

```ini
[Unit]
Description=Oxmgr daemon
After=network.target

[Service]
Type=simple
ExecStart=/home/ubuntu/.cargo/bin/oxmgr daemon run
Restart=always
RestartSec=2

[Install]
WantedBy=default.target
```

`WantedBy=default.target` (not `multi-user.target`) because it is a user unit. It needs
`sudo loginctl enable-linger ubuntu` to survive logout.

Note `ExecStart` uses the absolute path `/home/ubuntu/.cargo/bin/oxmgr`, and the app
`command` uses the absolute path `/home/ubuntu/.bun/bin/bun`. Nothing in this chain gets
your interactive PATH. See `06-shell-env.md`.

## Honest framing for the chapter

Recommending an obscure Rust tool to first-years is not the point. The point is what a
process manager gives you on top of systemd, and that the config format is more or less
the same idea in PM2, oxmgr, or a systemd unit with a watchdog. Present it as "here is
what I run and why", with PM2 named as the thing most tutorials will hand them, and the
concepts (health command, restart policy, declared environment, log routing) as the
transferable part.

## Drift found

`oxfile.toml.template` defines **two** apps, the site and an admin service on a second
port. The live `oxfile.toml` has only the first. `admin/server.tsx` is committed and
documented and nothing listens on 2057. Real, current, half-landed feature, and a good
illustration of templates drifting from reality. It pairs with the wildcard-shadowing
problem in `11-subdomains`.
