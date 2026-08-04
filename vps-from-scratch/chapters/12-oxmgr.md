# 12. the process is running and the site is down

```
$ sudo systemctl status myapp
● myapp.service - My web app
     Active: active (running) since Mon 2026-07-11; 3 weeks ago
```

Green. Three weeks of uptime. And the site returns 502.

systemd knows whether your process **exists**. It has no idea whether your process
**works**. A Node app that has deadlocked its event loop, exhausted its database
connection pool, or is returning 500 to every request is `active (running)` forever, and
systemd will happily report green while your site is completely down.

That gap is what a process manager fills.

## What a process manager adds

The tools in this category (PM2, supervisor, oxmgr, and others) give you the same handful
of things:

- **Health checks.** Poll an endpoint; restart when it stops answering.
- **Declared environment.** Variables in a config file next to the code, not scattered.
- **Log routing.** Per-app stdout and stderr files without touching journald.
- **Multiple apps in one config file**, versioned in the repo.
- **A status view.** One command that shows every service, its uptime, its memory, its
  restart count.

This box uses **oxmgr**, a process manager written in Rust. Fair warning: it is obscure.
Most tutorials will hand you PM2, and PM2 is a perfectly good answer. The concepts below
transfer directly; oxmgr even reads PM2 `ecosystem.config.js` files and can convert them.
What matters is understanding what the layer is for.

```bash
cargo install --git https://github.com/Vladimir-Urik/OxMgr
```

## The config

`oxfile.toml`, living in the application repo:

```toml
version = 1

[defaults]
restart_policy = "on_failure"
max_restarts = 5
restart_delay_secs = 5
stop_timeout_secs = 5

[[apps]]
name = "raghav56-tech"
command = "/home/ubuntu/.bun/bin/bun run --hot server.js"
cwd = "/home/ubuntu/raghav/raghav56.tech"

health_cmd = "curl -fsS http://127.0.0.1:2056/health"
health_interval_secs = 10
health_timeout_secs = 2
health_max_failures = 3

[apps.env]
NODE_ENV = "production"
PORT = "2056"
HOSTNAME = "127.0.0.1"

[apps.logs]
stdout = "/var/log/raghav56/raghav56.log"
stderr = "/var/log/raghav56/raghav56-error.log"
```

## The health check is the point

```toml
health_cmd = "curl -fsS http://127.0.0.1:2056/health"
health_interval_secs = 10
health_timeout_secs = 2
health_max_failures = 3
```

Every 10 seconds, hit `/health`. Give it 2 seconds to answer. After 3 consecutive
failures, restart the process.

Thirty seconds from wedged to restarted, without anyone noticing. This is the same idea
Kubernetes calls a `livenessProbe`, and it is the thing that turns "the process exists"
into "the process works".

`curl -fsS` matters: `-f` makes an HTTP error status a non-zero exit code, so a 500
counts as a failure rather than a success that happened to return text.

Your `/health` endpoint should be cheap and honest. Two lines is fine:

```js
if (url.pathname === "/health") return new Response("ok");
```

Do not make it query the database on every check unless you want a slow database to
trigger a restart loop. And do not make it *always* return ok regardless of state, which
is the opposite failure and surprisingly common.

## The environment section, and three traps

The comment above it in the real file says everything:

```toml
# Env vars must be explicit — OxMgr daemon runs in a clean env,
# Bun does NOT auto-load .env when launched by a process manager.
# HOSTNAME is set explicitly to override the system hostname env var.
```

**Trap 1: a supervisor daemon has a clean environment.** Nothing from your `.bashrc`
reaches it. Chapter 03 covered this; this is where it bites in practice. It is also why
`command` spells out `/home/ubuntu/.bun/bin/bun` in full rather than saying `bun`.

**Trap 2: do not rely on automatic `.env` loading.** Bun loads `.env` when you run it
yourself. Under a supervisor the working directory and the runtime's behaviour are things
you are now depending on invisibly. Declare the variables where you can see them.

**Trap 3: `HOSTNAME` is already taken.** The system sets it to the machine's hostname,
`oracler` on this box. An app doing `process.env.HOSTNAME || "127.0.0.1"` to choose a bind
address will try to bind to `oracler` and produce an error that makes no sense at all. The
explicit override exists because this happened.

That third one generalises: check whether the system already owns a variable name before
you use it. `HOSTNAME`, `USER`, `HOME`, `PATH`, `SHELL`, `LANG` are all taken.

## Two layers of supervision

```
systemd (system)          →  caddy
systemd (user) + linger   →  oxmgr daemon  →  bun run --hot server.js
```

The process manager itself needs supervising, and systemd does that:

```ini
# ~/.config/systemd/user/oxmgr.service
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

```bash
systemctl --user enable --now oxmgr
sudo loginctl enable-linger ubuntu     # do not skip this
```

Without `enable-linger`, the whole thing dies the moment you log out. It is a user
service; user services follow your session unless you say otherwise.

Not redundant, layered: systemd survives reboot and keeps the daemon alive; the daemon
does per-app health checking and log routing.

## Day to day

```bash
oxmgr list                    # status of everything
oxmgr logs raghav56-tech      # tail
oxmgr reload raghav56-tech    # restart after a deploy
oxmgr status raghav56-tech
```

```
$ oxmgr list
+----+---------------+---------+--------+---------+---------+----------+---------+---------+
| ID | NAME          | STATUS  | MODE   | PID     | UPTIME  | RESTARTS | RAM(MB) | HEALTH  |
+----+---------------+---------+--------+---------+---------+----------+---------+---------+
| 13 | raghav56-tech | running | single | 2426420 | 23d 21h | 0        | 53      | healthy |
+----+---------------+---------+--------+---------+---------+----------+---------+---------+
```

23 days, zero restarts, 53 MB resident, and a `HEALTH` column that means something
because a real check is behind it. That column is the entire reason this layer exists.

The deploy script's restart step is one line:

```bash
cmd_bun() {
    log "reloading bun..."
    oxmgr reload raghav56-tech
    ok "bun reloaded"
}
```

## Do you need this?

Honestly: not on day one.

systemd alone gets you restart-on-crash, start-on-boot, and logs. That is most of the
value. Add a process manager when one of these is true:

- You are running more than two or three services and want one status view.
- You have been bitten by a process that was running and not working.
- You want service definitions versioned in the repo alongside the code.

If you do add one, the concepts are what matter, not the tool: **a health command, a
restart policy, a declared environment, and log routing.** Write those four things down
for every service you run, whether they end up in an `oxfile.toml`, a PM2 ecosystem file,
or a systemd unit with `Type=notify` and a watchdog.

## A note on drift

This box's committed config template defines **two** apps: the site, and an admin service
on a second port. The live config has one. The admin service is written, committed, and
documented, and nothing is running it.

That is real, it is current, and it pairs with the wildcard problem from chapter 11: the
`*.raghav56.tech` block silently serves the main site at `admin.raghav56.tech` instead of
404ing, so the half-finished feature is invisible rather than obviously broken.

Templates drift from reality. When they do, the wildcard hides it. Both halves of that
sentence are worth remembering.

Next: deploying without SSHing in and remembering nine commands.
