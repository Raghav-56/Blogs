# Research: systemd units

## What is enabled

`systemctl list-unit-files --type=service --state=enabled` returns 49 units. The
non-distro ones: `alloy`, `caddy`, `docker`, `<teamapi>-backend`, `grafana-server`,
`loki`, `tailscaled`, `netfilter-persistent`, plus Oracle's
`snap.oracle-cloud-agent.*` and `unified-monitoring-agent*`.

Only two genuinely hand-written unit files live in `/etc/systemd/system/`. Everything
else in that directory is a symlink into `/usr/lib/systemd/system/`, which is the
distinction people miss: packages install to `/usr/lib`, you write to `/etc`, and `/etc`
wins.

## The first unit ever written on this box

`/home/ubuntu/raghav/dprc/raghav56.tech.service`, kept as an artifact rather than
installed. This is the genuine starting point of the whole story.

```ini
[Unit]
Description=Website raghav56.tech
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/raghav/raghav56.tech
EnvironmentFile=/home/ubuntu/raghav/raghav56.tech/.env

ExecStart=/usr/bin/node server.js

Restart=on-failure
RestartSec=5

# Capabilities needed to bind to privileged ports like port 80
# AmbientCapabilities=CAP_NET_BIND_SERVICE
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE

StandardOutput=append:/var/log/raghav56/raghav56.log
StandardError=append:/var/log/raghav56/raghav56-error.log

[Install]
WantedBy=multi-user.target
```

Note the commented-out capability lines. That is the exact moment of learning "a
non-root process cannot bind port 80", and the two ways out of it: grant the capability,
or put a proxy in front. This box eventually chose the proxy.

## The anti-pattern unit (anonymized)

A teammate's Node API, installed as a system service:

```ini
[Unit]
Description=Team API (nodemon dev)
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/<project>/backend
EnvironmentFile=/home/ubuntu/<project>/backend/.env
ExecStart=/usr/bin/npm run dev
Restart=on-failure
RestartSec=5
StandardOutput=append:/var/log/<project>/backend.log
StandardError=append:/var/log/<project>/backend-error.log

[Install]
WantedBy=multi-user.target
```

Three problems in eight lines, all verifiable:

1. **`ExecStart=/usr/bin/npm run dev`.** The live process tree is
   `npm run dev` → `sh -c "nodemon src/index.js"` → `nodemon` → `node src/index.js`.
   systemd is watching npm, four processes away from the thing that actually serves
   traffic. If node crashes, nodemon restarts it and systemd never notices, so
   `Restart=on-failure` and `systemctl status` are both telling you about the wrong
   process. Also, nodemon is a filesystem watcher meant for development, running in
   production.
2. **Cross-user `WorkingDirectory`.** `User=ubuntu` but the tree is owned by another
   account, so git refuses to operate on it: `fatal: detected dubious ownership`.
3. Combined with `app.listen(PORT)` and an open firewall port, this service is directly
   reachable on the public IP. See `05-network.md`.

The fix in one line: `ExecStart=/usr/bin/node src/index.js`, with `User=` matching the
directory owner. systemd is the process supervisor; you do not need npm to be one too.

## The one other hand-written system unit

```ini
# /etc/systemd/system/loki.service
[Unit]
Description=Loki service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=loki
ExecStart=/usr/bin/loki -config.file /etc/loki/config.yml
# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec = 120
Restart = on-failure
RestartSec = 2

[Install]
WantedBy=multi-user.target
```

`After=network-online.target` plus `Wants=network-online.target` is the correct pair when
a service genuinely needs a routable address. `After=network.target` alone (used by the
two units above) only means "after the networking stack started", not "after an IP
exists".

## User units

`~/.config/systemd/user/oxmgr.service`:

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

`~/.config/systemd/user/hermes-gateway.service` is the other one, and it is a good
example of a fuller unit: `RestartForceExitStatus=75`, `RestartPreventExitStatus=78`,
`KillMode=mixed`, `ExecReload=/bin/kill -USR1 $MAINPID`, `ExecStopPost=` cleanup, and
`TimeoutStopSec=60`.

Key differences from system units:

- Managed with `systemctl --user`, not `sudo systemctl`.
- Installed to `default.target`, not `multi-user.target`.
- **They die when you log out** unless linger is enabled:
  `sudo loginctl enable-linger ubuntu`. The 23-day process uptime under oxmgr proves it
  is enabled here.

## No cron at all

```
$ crontab -l
crontab: command not found
```

The `cron` package is not even installed. `/etc/cron.d/` holds only `certbot` and
`e2scrub_all` from packages. Every scheduled and supervised thing on this box runs
through systemd timers and units. That is a deliberate position worth stating: timers
give you logs in `journalctl`, dependency ordering, and `systemctl list-timers` to see
what runs next. Cron gives you a mail spool.

## Command crib for the chapter

```bash
sudo systemctl daemon-reload          # after editing any unit file
sudo systemctl enable --now myapp     # start it and start it at boot
sudo systemctl status myapp
journalctl -u myapp -f                # follow logs
journalctl -u myapp --since "10 min ago" -p err
systemctl --user status oxmgr         # user units
systemctl list-timers                 # what is scheduled
systemd-analyze verify ./myapp.service
```

`enable` and `start` are different verbs. `enable` without `--now` means "at next boot,
but not right now", which is the source of the classic "it works until I reboot" and its
mirror "it works now but died after reboot".
