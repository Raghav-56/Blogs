# 04. the naive first deploy

You have a server, a runtime, and an app. Time to put it online.

```bash
$ bun run server.js
Listening on port 3000
```

In another terminal, `curl localhost:3000` works. You are elated. Then:

- You close your laptop. The site goes down.
- You want it on port 80 so people do not have to type `:3000`. Permission denied.
- It crashes at 3am. Nobody restarts it.

Every one of these has an obvious wrong fix that mostly works, and that is the problem.

## Wrong fix one: nohup

```bash
nohup bun run server.js > app.log 2>&1 &
```

Worth breaking apart, because most people copy this without reading it:

| Part | Meaning |
|---|---|
| `nohup` | ignore `SIGHUP`, the signal your shell sends its children when it exits |
| `> app.log` | send stdout to a file instead of the terminal |
| `2>&1` | send stderr (fd 2) to wherever stdout (fd 1) is now going |
| `&` | run in the background, give me my prompt back |

It genuinely works. It also means: no log rotation (`app.log` grows until the disk is
full), no restart on crash, no restart on reboot, and no way to find the process later
except `ps aux | grep`.

And you will run it twice.

### The PID story, which is worth the detour

The obvious next step is to save the PID so you can stop it later:

```bash
nohup bun run server.js > app.log 2>&1 & echo $! > app.pid
```

`$!` is "the PID of the last background job". On my earlier server this produced a PID
that did not match what `ps` showed, and I lost an evening to it. The explanation I was
given, confidently, was that `nohup` forks a child, so `$!` captures the wrapper rather
than the real process.

That is wrong, and you can check it in ten seconds:

```
$ nohup sleep 40 >/dev/null 2>&1 &
$ echo $!
487308
$ ps -o pid=,comm= -p 487308
 487308 sleep
```

`$!` **is** the process. `nohup` does not fork. It sets `SIGHUP` to ignore, redirects
output, and then `execve`s your command, replacing itself in the same process. Same PID
start to finish. The giveaway in the bad explanation was a process tree where the parent
had a *higher* PID than its child, which cannot happen.

`setsid` genuinely does fork, and that is a real difference:

```
$ setsid sleep 41 >/dev/null 2>&1 </dev/null &
$ echo $!
487317
$ ps -o pid=,comm= -p 487317      # empty. already gone.
$ ps -eo pid=,args= | grep 'sleep 41'
487319 sleep 41
```

So one explanation was right and the other was wrong, delivered with identical
confidence. **Check the claim against your own box.** It is usually one command.

What was actually wrong on my server, visible in the timestamps I had pasted and that
neither of us read: the PID `ps` was showing had started an hour earlier, on a different
terminal. It was a leftover from a previous run still holding the port, and the process I
had just launched had died immediately on `EADDRINUSE`. Two copies of the app, and the
whole PID discussion was chasing a symptom.

The workaround I shipped was `sleep 1; pgrep -f "my app" > app.pid`, which is worse than
the bug: `pgrep` finds the *stale* process and writes its PID to the file, so now the PID
file confidently points at the wrong instance.

The real lesson is not about `nohup`. It is that once you are hand-rolling PID files, you
have started writing a bad process manager, and there is a good one already installed.

## Wrong fix two: tmux

```bash
tmux new -s app
bun run server.js
# Ctrl-b d
```

Better, because you can attach and read the output. Still dies on reboot, still no
restart on crash, and now your production service is a thing that exists only in the
memory of one long-lived shell session. Nobody else on the machine knows it is there.

tmux is right for a long build. It is not a way to run a service.

## Wrong fix three: run it on port 80

```bash
$ PORT=80 bun run server.js
error: Failed to start server. Is port 80 in use?
EACCES: permission denied
```

Ports below 1024 are privileged. Only root, or a process holding the
`CAP_NET_BIND_SERVICE` capability, can bind them. This is an old Unix rule from when
having root on a machine meant something about who you were, and it survives because it
is still a useful boundary.

So you do this:

```bash
sudo bun run server.js
```

And it works, and this is the worst outcome in the chapter, because now every line of
your application, every dependency, every transitive package you have never read, runs as
root. A path traversal bug in your file server is now a whole-machine compromise instead
of an annoyance.

## The archaeology

Here is the first service file ever written on this box. It is kept in a folder called
`dprc/` as an artifact rather than being installed:

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

Look at the commented-out capability lines. That is the exact moment of discovering the
port 80 problem, writing down the fix, and then not using it.

Not using it turned out to be right, and the reason is the next section.

## The two real answers

**Answer A: grant the capability.** systemd can hand a specific privilege to a specific
service without making it root:

```ini
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

Now the process runs as `ubuntu`, and can bind port 80, and can do nothing else special.
This is genuinely fine, and it is exactly what Caddy's own service file does:

```ini
User=caddy
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
```

That is how a web server running as an unprivileged user holds port 80 on this machine
right now.

**Answer B: do not bind port 80 at all.** Run your app on a high port, on loopback only,
and put a reverse proxy in front of it. The proxy holds 80 and 443; your app holds
`127.0.0.1:2056` and is not reachable from outside the machine at all.

This box chose B, and the reason is not really about port 80:

- **One app cannot hold port 443 for five different sites.** As soon as you have a second
  service, you need something in front that routes by hostname.
- **TLS in one place.** One certificate, one renewal, one configuration. Not per-app TLS
  in five different frameworks.
- **Your app never faces the internet.** It does not have to be hardened against
  malformed requests, slowloris, or header smuggling. Something purpose-built does that.
- **Deploys stop being outages.** Restart the app; the proxy holds the connection and
  fails a few requests instead of dropping the port.

Chapters 07 and 10 build that. This chapter's job is to explain why you would want it.

## The thing you have to internalise now

Your app should bind `127.0.0.1`, not `0.0.0.0`.

Loopback-only means the kernel will not deliver a packet from the network to your process
at all, no matter what your firewall says. It is the strongest possible statement of "only
things on this machine may talk to this". Then exactly one process, the proxy, is exposed,
and it is the one written to be exposed.

On this box:

```
$ ss -tlnp
LISTEN  127.0.0.1:2056   bun      ← the app, unreachable from outside
LISTEN          *:443    caddy    ← the only public door
```

Chapter 06 goes into this properly, with the two counter-examples from this same machine.
For now: **`127.0.0.1` in development, `127.0.0.1` in production, and a proxy in front.**

## What still is not solved

Even after you fix port 80, `bun run server.js` in a tmux session is not a deployment:

- It does not start on boot.
- It does not restart when it crashes.
- Its output goes nowhere useful.
- Nothing on the machine can tell you whether it is supposed to be running.

That is a solved problem, and the solution is already installed on your machine.

Next: systemd.
