# Research: what the old chat logs prove

Source: four ChatGPT exports in `../Chat_exports/`, spanning **Nov 2025 to Aug 2026**.
Two of the five files are byte-identical duplicates. They are incomplete and were not
written as documentation, but they contain something the live box cannot give: **the
errors, in order, with timestamps.**

> ⚠️ **These exports contain a live bcrypt hash** for `dev.raghav56.tech` basic auth
> (username `raghav`), pasted verbatim while debugging. Never quote them without
> redaction. See the warning at the bottom of this file.

---

## 1. There was an earlier server

The Nov 2025 transcript is a **different machine**:

| | Nov 2025 box | Current box |
|---|---|---|
| Hostname | `oracle-rg` | `oracler` |
| Interface | `ens3` | `enp0s6` |
| Private IP | `10.0.0.228/24` | `10.0.0.22/24` |
| Task limit | 1073 (small shape) | 4 OCPU / 23 GiB |
| Project | `Agent_kdg`, FastAPI + `uv` | Astro + Bun |

So the progression in this series is not reconstructed. It happened twice, on two
machines, seven months apart, and the second time went much faster.

The site repo's first commit is 2026-06-28. The Nov 2025 transcript predates it by more
than half a year.

---

## 2. Real systemd failures, verbatim

This is the material chapter 05 was missing. Every one of these is a real terminal paste.

**Attempt 1: relative paths.** The unit as first written:

```ini
[Service]
User=ubuntu
WorkingDirectory=~/raghav/Agent_kdg
ExecStart=uv run main.py
StandardOutput=append:logs/server.log
StandardError=append:logs/errors.log
```

Four things wrong. The failure:

```
Failed to start agent.service: Unit agent.service has a bad unit file setting.

× agent.service - Agent_kdg FastAPI Server
     Loaded: bad-setting (Reason: Unit agent.service has a bad unit file setting.)
     Active: failed (Result: exit-code)
   Main PID: 3476 (code=exited, status=203/EXEC)

systemd[1]: /etc/systemd/system/agent.service:6: WorkingDirectory= path is not absolute: ~/raghav/Agent_kdg
systemd[1]: agent.service: Unit configuration has fatal error, unit will not be started.
```

Three exact strings worth putting in the blog because people search for them:

- `WorkingDirectory= path is not absolute: ~/...`: systemd does not expand `~`. It is a
  shell feature, and there is no shell.
- `status=203/EXEC`: systemd could not execute the binary. Nearly always a wrong path,
  a missing file, or a missing execute bit.
- `Loaded: bad-setting`: the unit file itself is invalid, so it was never even attempted.

**Attempt 2: enabled and started, but no process.**

```
$ sudo systemctl enable agent
Created symlink /etc/systemd/system/multi-user.target.wants/agent.service → /etc/systemd/system/agent.service.
$ sudo systemctl start agent
$ pgrep -f "uv run main.py"
$          # nothing
```

`enable` and `start` both reported success and nothing was running. The `enable` output is
worth showing on its own, because it makes visible what `enable` actually does: it creates
a symlink into the target's `.wants/` directory. That is the whole mechanism.

**The working version:**

```ini
[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/raghav/Agent_kdg
ExecStart=/home/ubuntu/.local/bin/uv run main.py
Environment="PATH=/usr/local/bin:/usr/bin:/bin:/home/ubuntu/.local/bin"
Restart=always
RestartSec=5
```

And it ran for four days:

```
Active: active (running) since Thu 2025-11-13 09:30:17 UTC; 4 days ago
   Main PID: 537 (uv)
     CGroup: /system.slice/agent.service
             ├─537 /home/ubuntu/.local/bin/uv run main.py
             └─634 /home/ubuntu/raghav/Agent_kdg/.venv/bin/python3 main.py
```

That `CGroup` tree is a good illustration for the `npm run dev` anti-pattern: systemd
tracks the whole cgroup, but `Main PID` is the wrapper (`uv`, 537), not the process
actually serving requests (`python3`, 634).

---

## 3. The nohup / setsid PID puzzle, and where the AI got it wrong

The transcript spends a long time on this:

```
$ nohup uv run main.py > logs/server.log 2>&1 & echo $! > logs/server.pid
[1] 3305
$ ps aux | grep uv 
ubuntu  3166  0.0  3.9 240100 38468 pts/2  Sl+  14:00  0:00 uv run main.py
$ cat server.pid 
3305
```

The AI's explanation was that `nohup` forks a child, so `$!` captures the wrapper rather
than the app, and it drew a process tree showing `nohup(3305) → uv(3166)`.

**That is wrong, and it is checkable in ten seconds:**

```
$ nohup sleep 40 >/dev/null 2>&1 &
$ echo $!
487308
$ ps -o pid=,comm= -p 487308
 487308 sleep
```

`$!` **is** the sleep process. `nohup` does not fork. It sets `SIGHUP` to ignore,
redirects output, and then `execve`s the command, replacing itself. Same PID throughout.
A parent cannot have a lower PID than its child anyway, which is the tell: the AI's tree
had `nohup(3305)` as the parent of `uv(3166)`.

`setsid` genuinely does fork, and only when the caller is already a process group leader
(which a background job is):

```
$ setsid sleep 41 >/dev/null 2>&1 </dev/null &
$ echo $!
487317
$ ps -o pid=,comm= -p 487317      # empty, it already exited
$ ps -eo pid=,args= | grep 'sleep 41'
487319 sleep 41
```

So the AI's `setsid` explanation was right and its `nohup` explanation was wrong, while
sounding equally confident about both.

**What was actually happening in the transcript:** look at the timestamps. PID 3166
started at **14:00** on **pts/2**. The `nohup` command was typed at roughly **15:07** on a
different terminal, and the `grep` on the same line is PID 3319. So 3166 is a leftover
process from an earlier session that was still holding port 8086, and the newly launched
3305 had already died, almost certainly on `EADDRINUSE`. The real bug was two copies of
the app, and neither the question nor the answer noticed.

The fix that got shipped was `sleep 1; pgrep -f "uv run main.py" > logs/server.pid`, which
papers over it: `pgrep` returns the *stale* process and writes its PID to the file, so the
PID file now confidently points at the wrong instance.

This is the best "verify the advice" example in the entire archive.

---

## 4. The 403, and where privilege separation actually came from

July 2026, first attempt at serving the terminal site. Caddy config had:

```caddyfile
root * /home/ubuntu/raghav/raghav56.tech/
```

Result:

```
$ curl -v https://raghav56.tech
< HTTP/2 403
< server: Caddy
< x-request-id: {http.request.id}
< content-length: 0
```

A 403 with **zero content length** and no explanation. TLS fine, routing fine, file
present and world-readable.

The diagnostic that solved it is the single most useful command in the archive:

```
$ namei -l /home/ubuntu/raghav/raghav56.tech/static/index.txt
f: /home/ubuntu/raghav/raghav56.tech/static/index.txt
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu          ← 750, caddy is not in group ubuntu
drwx------ ubuntu ubuntu raghav          ← 700, nobody but ubuntu
drwxrwxr-x ubuntu ubuntu raghav56.tech
drwxrwxr-x ubuntu ubuntu static
-rw-rw-r-- ubuntu ubuntu index.txt       ← world-readable, and irrelevant
```

`ls -l index.txt` says `rw-rw-r--`, world readable. It lies, because **directory
permissions compose**: to read a file you need execute (traverse) on every directory above
it. Two levels up, `/home/ubuntu/raghav` is `700`.

Then the important bit, in Raghav's own words:

> yes it doesnt and obv im not going to give caddy those permissions
>
> go through the original goal again, is this really the correct way to do things?

and, to a suggested workaround a few messages later:

> i dont like what youre suggesting

The `700` source tree plus `755` public web root plus rsync-across-the-boundary design in
chapter 14 was **not designed up front**. It was arrived at by hitting a 403 and refusing
the easy fix of loosening the home directory. That is a much better story than the tidy
table in `docs/overview.md` suggests.

---

## 5. A real placeholder bug

Look again at that curl output:

```
< x-request-id: {http.request.id}
```

The header contains the **literal string** `{http.request.id}`, not a request ID. The
config said:

```caddyfile
header {
    X-Request-ID {http.request.id}
}
```

There is no such Caddy placeholder. The correct one is `{http.request.uuid}`, which is
what the live config uses today. An unknown placeholder is not an error; it is emitted
verbatim.

So for some period the site served a request-ID header that was the same constant string
on every response, and nothing anywhere complained. Chapter 15 recommends this header;
it needs the warning attached.

Check yours: `curl -sI https://yoursite/ | grep -i request-id`.

---

## 6. The process manager config drifted

The July 2026 `oxfile.toml` versus today's:

```toml
# then                                  # now
restart_policy = "on-failure"           restart_policy = "on_failure"   (underscore)
env_file = ".env"                       (gone, env declared inline)
wait_ready = true                       (gone)
ready_timeout_secs = 30                 (gone)
stdout = "/var/log/..."                 [apps.logs] stdout = "..."
stderr = "/var/log/..."                 [apps.logs] stderr = "..."
# max_memory_mb, max_cpu_percent, cgroup_enforce commented out in both
```

Hyphen to underscore in an enum value, keys removed, keys moved into a subsection. Five
breaking changes in one config file in about three weeks. This is the honest cost of
running a small, fast-moving tool, and it belongs in chapter 12 next to the
recommendation.

Also note `wait_ready = true` / `ready_timeout_secs = 30`: the old config had a
**readiness gate**, where the new process had to pass the health check before the old one
was stopped. That is zero-downtime reload, and it is gone from the current config.

---

## 7. Advice that was right then and wrong now

The Nov 2025 transcript repeatedly says:

```bash
sudo ufw status
sudo ufw allow 5000/tcp
```

`ufw` is not installed on this Oracle image and never was. Every one of those commands
would have returned `command not found`.

It also says, correctly for the time:

> ✅ **Fix: Bind to All Interfaces** ... `host="0.0.0.0"`

At that point there was no reverse proxy. `0.0.0.0` was the only way to reach the app, and
the advice was right. Chapter 06 of this series says the opposite, and is also right,
because by then Caddy exists and terminates everything.

**Same question, opposite correct answers, depending on what else is running.** That
nuance has to be in the series, or a reader on day one will follow chapter 06, bind
loopback, have no proxy, and be unable to reach anything.

The Aug 2026 conversation shows the model updated too, unprompted:

> Your backend can safely continue listening on `127.0.0.1`; Nginx handles all incoming
> public traffic and forwards it internally.

---

## 8. The teaching transcript

The Aug 2026 export is a voice conversation (Hinglish) of Raghav explaining this stack to
someone else. Analogies that came out of it, in his own framing:

- **Nginx is a receptionist / ticket administrator.** The public IP is the building
  entrance, Nginx is the front desk, backends are departments. Nobody walks into a
  department from the street.
- **DNS is a pyramid of routers**, resolved step by step from the top.
- The line that lands it:

  > The backend never became public.

- And the closing:

  > Not magic. Just layers.

That conversation ends with him asking for it to be turned into a blog post, and getting
an outline instead of a post. This series is the thing that request was asking for.

Also captured: the misconception, and its correction.

> So you are saying like Nginx decrypt the response and browser encrypt it. Nahi, no,
> decrypt, nothing, nothing, it just passes on.

TLS terminates at the proxy when the certificate lives there. The proxy decrypts the
request, forwards it in plaintext over loopback, and re-encrypts the response. It is not a
passive pipe. Worth stating precisely, because the intuition that it "just passes on" is
the common one.

---

## 9. Things planned and still not built

From the original SRS pasted in July 2026:

- **`ssh raghav56.tech`** dropping straight into a TUI, no shell. Phases 4 and 5. Not
  built.
- An AI chatbot that answers questions about the work. Not built. Still listed as "Phase
  4" in the repo docs today.
- `curl raghav56.tech/resume.pdf` 302-redirecting to `raw.githubusercontent.com`. Shipped
  differently: the PDF is built from a git submodule and served locally.
- `glow -w 78` into a `static/` directory. Shipped as `-w 100` for pages and a hand-drawn
  54-column box for the card, into `/var/www`.

Design drift between an SRS and what ships is normal. Naming it is more useful than
pretending the plan was followed.

---

## ⚠️ Redaction required

`Chat_exports/ChatGPT-Terminal Website Setup.md` contains, pasted three times while
debugging, the **live bcrypt hash** for `dev.raghav56.tech` basic auth together with the
username and the hostname. That hash is still in
`/etc/caddy/sites-enabled/raghav56.tech.caddy` on the server.

The `Blogs` repository is **public**. Nothing from these exports may be quoted into a
chapter without redaction, and the credential itself needs rotating regardless of what
this series does.

Everything quoted in this file has been redacted or is non-sensitive.
