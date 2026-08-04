# 05. systemd

Your app crashed at 3am. You found out at 11am, from a friend.

There is a program on your machine whose entire job is to notice that and fix it. It
started every other service on the box. It is called systemd, it has a reputation for
being complicated, and the part you need fits on one screen.

## A unit file

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My web app
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
EnvironmentFile=/home/ubuntu/myapp/.env
ExecStart=/home/ubuntu/.bun/bin/bun run server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Line by line, because every one of these is a decision:

**`After=network-online.target` / `Wants=network-online.target`.** Both, together. This
is a real distinction that trips people up. `network.target` means "the networking stack
has been set up", which is not the same as "an IP address exists". If your service binds
a specific address at startup, it needs `network-online.target`, and `Wants=` is what
actually pulls that target in. This box's Loki unit gets it right; two other units on the
box use `After=network.target` alone, and they get away with it only because they bind
`0.0.0.0` or loopback.

**`Type=simple`** means "the thing I exec is the service". Use this unless your program
forks into the background on purpose (then `Type=forking`) or explicitly tells systemd
when it is ready (`Type=notify`, which is what Caddy uses).

**`User=ubuntu`.** Without this, your service runs as root. There is almost never a
reason. Make sure the user owns `WorkingDirectory`, which is a mistake you will see
committed shortly.

**`ExecStart` must be an absolute path**, and takes no shell. No pipes, no `&&`, no `$VAR`
expansion, no globs. If you need shell syntax, either put it in a script and exec the
script, or write `ExecStart=/bin/bash -c '...'` and feel appropriately bad about it. See
chapter 03 for why the path to your runtime has to be spelled out in full.

**`Restart=on-failure`** restarts on a non-zero exit or a signal, but not on a clean
`exit 0`. That is usually what you want: `Restart=always` will fight you when you
deliberately stop something. `RestartSec=5` waits five seconds between attempts, so a
service that crashes instantly on startup does not spin your CPU. systemd also has a
built-in rate limiter and will give up after five restarts in ten seconds, which is
correct: at that point it is broken, not unlucky.

**`WantedBy=multi-user.target`** is what makes `systemctl enable` do anything. It means
"start me at boot, in normal multi-user mode".

## Running it

```bash
sudo systemctl daemon-reload         # ALWAYS after editing a unit file
sudo systemctl enable --now myapp    # start now AND at boot
sudo systemctl status myapp
```

`enable` and `start` are different verbs, and mixing them up produces the two classic
bugs: `start` without `enable` means it works until you reboot, and `enable` without
`--now` means it does nothing until you reboot.

`daemon-reload` after every edit. Without it systemd is still running your old file and
you will conclude the change did nothing.

## Logs

```bash
journalctl -u myapp -f                      # follow
journalctl -u myapp --since "10 min ago"    # recent
journalctl -u myapp -p err                  # errors only
journalctl -u myapp -b                      # since last boot
```

You do not have to configure logging. Anything your app writes to stdout or stderr goes
to the journal, tagged, timestamped, and rotated. Write to stdout and stop thinking about
it.

If you want plain files instead:

```ini
StandardOutput=append:/var/log/myapp/app.log
StandardError=append:/var/log/myapp/error.log
```

Create the directory first and make sure `User=` can write it. systemd will not create it
for you, and the failure is silent-ish.

## The counter-example

This box also runs a Node API that a teammate and I put up. Its unit is a good teacher,
because it works, and it is wrong in three ways at once. Anonymized, otherwise verbatim:

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
...
```

**Problem 1: `ExecStart=/usr/bin/npm run dev`.** Here is the live process tree:

```
npm run dev
 └─ sh -c "nodemon src/index.js"
     └─ nodemon
         └─ node src/index.js      ← the actual server
```

systemd is supervising `npm`. The thing serving traffic is four processes away. If node
crashes, nodemon quietly restarts it and systemd never learns anything happened. If node
wedges, systemd sees npm alive and reports `active (running)`. `Restart=on-failure` is
watching the wrong process. `systemctl status` is telling you about the wrong process.
Every restart guarantee in the file is void.

The description field even admits it: "nodemon dev". Nodemon is a filesystem watcher for
development. In production it is a memory-resident process whose only job is to restart
your app when files change, on a machine where files do not change.

Fix: `ExecStart=/usr/bin/node src/index.js`. systemd is the supervisor. You do not need a
second one, and you definitely do not need `npm` to be one.

**Problem 2: `User=ubuntu` with a `WorkingDirectory` owned by someone else.** git refuses
to touch it: `fatal: detected dubious ownership`. Any deploy that involves a `git pull`
in that directory fails.

**Problem 3** is about network binding, and it is bad enough that it gets its own chapter.
Turn the page.

## systemd user units

You can run services as yourself, without sudo, in `~/.config/systemd/user/`. This box
does exactly that for its process manager:

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

Three differences from a system unit:

1. Manage it with `systemctl --user`, no sudo.
2. `WantedBy=default.target`, not `multi-user.target`.
3. **It dies when you log out**, unless you enable lingering:

```bash
sudo loginctl enable-linger ubuntu
```

That is a one-time command per user and it is not optional. Without it, your carefully
configured user service runs beautifully until you close your SSH session. The process
supervised through this unit has 23 days of uptime, which is the proof lingering is on.

Use user units for things that belong to you. Use system units for things that belong to
the machine.

## No cron on this box

```
$ crontab -l
crontab: command not found
```

The package is not installed, deliberately. Everything scheduled runs as a systemd timer
instead. You get logs in `journalctl`, dependency ordering, `systemctl list-timers` to see
exactly what runs next, and the ability to trigger a run by hand with
`systemctl start foo.service`. Cron gives you a mail spool nobody reads.

This matters later: when a certificate renewal is not happening, the first question is
"is it a timer or a cron job", and on a modern Ubuntu the answer is nearly always a timer.

```bash
systemctl list-timers
```

## Debugging crib

```bash
systemd-analyze verify ./myapp.service   # syntax check before installing
systemctl cat myapp                      # what systemd thinks the file says
systemctl show myapp -p Environment      # what env it will actually get
journalctl -u myapp -n 50 --no-pager
```

`systemctl cat` is the one to reach for when a change appears to have no effect. It shows
the file as loaded, including any drop-in overrides, and it will show you that you edited
a file in the wrong directory or forgot `daemon-reload`.

## What systemd still cannot tell you

It knows whether your process **exists**. It has no idea whether your process is
**working**.

A Node app that has deadlocked its event loop, or exhausted its database pool, or is
returning 500 to everything, is `active (running)` forever. systemd will happily report
green while your site is completely down.

That gap is what chapter 12 is about.

Next: the app says it is listening and nothing can reach it.
