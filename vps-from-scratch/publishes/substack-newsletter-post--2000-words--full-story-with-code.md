<!--
WHAT THIS FILE IS
Substack newsletter post. ~2,000 words. Paste the body below (everything under the
title) straight into the Substack editor.

Substack notes:
- No markdown tables anywhere in this file. Substack's editor mangles them on paste.
  Everything is bullets and bold lead-ins instead.
- Code blocks are short and few, because they render narrow on mobile.
- Suggested title and subtitle are at the top. Delete this comment block before pasting.
-->

**Title:** Your app is running. Nothing can reach it.

**Subtitle:** I put my site on a free server, and here is what broke at every step.

---

Every tutorial about deploying a website skips the part where it does not work.

I have now done this twice, on two different servers, seven months apart. The second time
went much faster, not because I had memorised the commands, but because I had collected
the failures. This is that collection.

Everything here is real. The configs are copied off a live Oracle Cloud box that has been
up for three months, and the error messages are pasted from my own terminal.

---

## The free machine is real, and it is not small

Oracle Cloud's free tier is not a trial. It does not expire. The ARM allocation is **4
CPUs and 24 GB of RAM**, permanently, for nothing.

For comparison, the six-dollars-a-month box everyone recommends is 1 CPU and 1 GB.

Here is my actual server:

```
$ nproc
4
$ free -h
               total        used        free   available
Mem:            23Gi       3.1Gi       686Mi        20Gi
$ uptime -p
up 12 weeks, 6 days
```

Four cores, 23 GB, three months of uptime. It runs a website, a Postgres database, a log
aggregator, a metrics dashboard, and a couple of other people's projects, and it is using
3 GB.

Two catches nobody mentions. You will get **"Out of capacity for shape
VM.Standard.A1.Flex"**, probably several times. That error is transient, not a rejection.
Retry at a different hour, pick a less popular home region at signup (you cannot change it
later), or create a smaller instance and scale it up. And your machine is ARM, so when you
download a binary, get the `arm64` build. An x86 binary on ARM gives you `Exec format
error` and no other clue.

---

## It runs. You close the laptop. It stops.

Your first deploy is `bun run server.js` in an SSH session. It works. Then your connection
drops and the site dies with it.

The obvious fixes are `nohup` and `tmux`. Both keep the process alive after you log out.
Neither restarts it when it crashes, neither survives a reboot, and neither gives anyone
else on the machine a way to know it exists.

The real answer is already installed. Here is roughly my first attempt at a systemd
service, from that earlier server:

```ini
[Service]
User=ubuntu
WorkingDirectory=~/raghav/Agent_kdg
ExecStart=uv run main.py
```

Four things wrong in three lines. systemd told me so:

```
Loaded: bad-setting
Main PID: 3476 (code=exited, status=203/EXEC)
WorkingDirectory= path is not absolute: ~/raghav/Agent_kdg
```

Three strings worth recognising, because you will meet all of them:

**`path is not absolute: ~/...`** means systemd does not expand `~`. Tilde expansion is a
shell feature and there is no shell here. Same for `$HOME`, globs, and `&&`.

**`status=203/EXEC`** means systemd could not execute what you named. Wrong path, missing
file, or missing execute bit.

**`Loaded: bad-setting`** means the file itself is invalid, so nothing ever ran. That is
different from `Loaded: loaded` plus `Active: failed`, which means your program started
and exited.

Then the quiet one that catches everybody exactly once:

```
$ sudo systemctl start agent
$ pgrep -f "uv run main.py"
$            # nothing. no error. no output.
```

Both commands succeeded and nothing was running, because `ExecStart` said `uv` and systemd
has no idea where that is. **systemd does not read your `.bashrc`.** Neither does cron.
Neither does `ssh host "command"`, which is how CI deploys. Every tool you installed into
your home directory is invisible to all three.

Write the path in full. `which uv` tells you what to paste. Mine has run untouched ever
since.

---

## It says it is listening and nothing can reach it

This is the one that wastes the most time, and it is not really about any tool.

Three separate systems have to agree before a packet reaches your process. They fail
independently, and they all fail silently.

**One: what address your app bound to.** A socket is bound to an address *and* a port.
`127.0.0.1:3000` means the kernel will only deliver packets that came from this machine.
Not "is blocked from" the internet. Cannot receive from it. `0.0.0.0` means every
interface, including the public one.

Read the address column, not the port:

```
$ ss -tlnp
LISTEN  127.0.0.1:2056   bun     ← unreachable from outside
LISTEN          *:443    caddy   ← the only public door
```

`app.listen(PORT)` with no host argument binds `0.0.0.0`. Every Express tutorial writes it
that way, and it is the most common accidental exposure in student projects.

**Two: the host firewall.** Every guide says `sudo ufw allow 80`. On the Oracle image
`ufw` is not installed and that command does not exist. Rules are raw iptables with a
default DROP policy. And whatever you do, never run `iptables -F` on Oracle Cloud: there
is a chain in there carrying cloud metadata, DHCP, NTP, and the iSCSI mount for your boot
volume. Flushing it can cost you the root disk.

**Three: the cloud firewall**, which is invisible from inside the machine. Oracle's
security list opens port 22 and nothing else by default. You can have everything on the
box correct and still get nothing, because the packet never arrived.

The diagnostic that identifies the layer instantly is the *shape* of the failure. A
request that hangs and then times out means a firewall dropped it, layer two or three. An
instant "connection refused" means the packet reached the machine and nothing was
listening, layer one. Connects but no response means your app is broken and this is not a
network problem at all.

One nuance I got wrong for a long time. Early on, with no reverse proxy, `0.0.0.0` is
correct and `127.0.0.1` will drive you mad, because nothing outside can reach it including
you. Once a proxy exists, every backend goes back to loopback. Same question, opposite
correct answers, depending on what else is running.

---

## Two apps, one port 443

Eventually a friend wants to host their project on your box too, and there is only one
port 443.

The way I ended up explaining a reverse proxy, which is the version that stuck:

**The proxy is a receptionist.** The public IP is the building entrance. The receptionist
sits at the front desk. Your backends are departments upstairs. Nobody walks in off the
street and straight into a department.

Which answers the question that confused me for months: if my backend is on `127.0.0.1`,
how is anyone on the internet using it?

They are not. They are talking to the receptionist. **The backend never became public.**

I started on nginx and moved to Caddy. Learn nginx anyway, because it is on every server
you will ever inherit, but know what it costs you. In nginx, these four lines are
mandatory and nobody warns you:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Miss the last one and your app thinks the request was plain HTTP, emits `http://` links on
an HTTPS page, and browsers block them as mixed content. That bug report makes no sense
when you receive it.

Here is the equivalent complete Caddy config, and this is a real file from my server:

```
api.example.com {
    reverse_proxy localhost:5001
}
```

That obtains a certificate, renews it forever, redirects HTTP to HTTPS, sets all four
headers, and turns on HTTP/2 and HTTP/3.

---

## Certificates are easy. Renewals are where you fail.

Getting HTTPS takes one command. Keeping it is the hard part, and a certificate that
stops renewing takes your site down 90 days after you stopped thinking about it.

On my box, right now, **both certificates have broken renewal**, for two different
reasons, and that is what makes them a good example.

One uses `authenticator = manual`, which means certbot stops and waits for a human to add
a DNS record. On a server at 3am, nobody answers.

The other renews through the nginx plugin. I switched to Caddy months ago and nginx is
stopped. **Renewal configs do not follow you when you change web servers.** They keep
pointing at the old one and fail quietly.

There is a third failure waiting even if renewal works: `/etc/letsencrypt/renewal-hooks/`
is empty by default, so nothing reloads your web server afterwards. The certificate on
disk is fresh and the one being served is expired, and every check you run on the file
says everything is fine.

Two things to do the day you set up TLS:

```bash
sudo certbot renew --dry-run          # surfaces all of the above, today
```

and write the reload hook. If you do not specifically need a wildcard certificate, use
Caddy and skip this entire section of your life.

---

## The 403 that redesigned my server

My first attempt at serving files pointed the web server straight at my project folder.
Every request returned:

```
HTTP/2 403
content-length: 0
```

A 403 with zero bytes of explanation. TLS fine. Routing fine. The file existed, and
`ls -l` said it was world readable.

The command that solves this in one line:

```
$ namei -l /home/ubuntu/raghav/site/static/index.txt
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu
drwx------ ubuntu ubuntu raghav          ← 700. there it is.
-rw-rw-r-- ubuntu ubuntu index.txt       ← world readable, and irrelevant
```

`namei -l` walks every component of a path and prints its permissions. **Directory
permissions compose:** to read a file you need traverse permission on every directory
above it. My file being readable meant nothing when a folder two levels up was `700`.

The obvious fix is `chmod 755` on your home directory. Do not. That makes every project,
key, and dotfile you own readable by every account on the machine, to fix one file.

I refused to loosen it, and that refusal is where my actual architecture came from: the
source tree stays at `700` and the web server genuinely cannot read it, the build copies
compiled output into a separate public folder, and the web server only ever sees HTML and
CSS. If a file server ever has a path traversal bug, it leaks pages that were already
public instead of my `.env` and my `.git` directory.

The 403 was not a problem to work around. It was the filesystem telling me the design was
wrong.

---

## Check the advice, including this post

While debugging that first server, I was told that `nohup` forks a child process, and that
this explained a PID mismatch I was seeing. It sounded reasonable. It was wrong.

`nohup` does not fork. It ignores the hangup signal, redirects output, and then replaces
itself with your command in the same process. You can verify it in ten seconds:

```
$ nohup sleep 40 >/dev/null 2>&1 &
$ echo $!
487308
$ ps -o pid=,comm= -p 487308
 487308 sleep
```

The PID is the process. Meanwhile `setsid`, which I was told was equivalent, genuinely
does fork.

And the actual bug, which neither the question nor the answer noticed: reading my own
pasted timestamps months later, the process I was looking at had started an hour earlier
on a different terminal. It was a leftover still holding the port, and the one I had just
launched had died immediately. Two copies of the app. The whole PID discussion was
chasing a symptom.

The fix I shipped writes the *stale* process ID into the file, confidently, forever.

So: check things against your own machine. It is usually one command, and the difference
between a confident explanation and a correct one is not visible from the outside.

---

## What you actually end up with

A machine that costs nothing. A domain. HTTPS that renews itself. A service that restarts
when it crashes. `git push` as the deploy command. And, more useful than any of it, enough
of a model of binding and firewalls and proxies to debug the next thing yourself.

Not magic. Just layers.
