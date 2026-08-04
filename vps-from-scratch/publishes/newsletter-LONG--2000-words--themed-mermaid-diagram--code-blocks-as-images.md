<!--
WHAT THIS FILE IS
Long-form newsletter post. ~2,000 words of prose. Written for a Canva layout where the
code blocks are rendered as images.

FOR THE DESIGNER
- 9 code blocks, each marked `<!-- IMAGE n -->` with an italic caption line underneath.
  They are deliberately short so they read as graphics, not walls of text. Longest line
  is 75 characters, so nothing wraps at any sensible width. Use a single monospace font
  and one background colour across all 9 so they read as a set.
- IMAGE 8 (`namei -l`) is the hero. If one code image gets full width, make it that one.
- IMAGE 1 is two words. Set it big, centred, as the opening visual.
- One mermaid diagram, marked `<!-- DIAGRAM -->`. It is already themed and coloured, so
  render it as-is at https://mermaid.live and drop in the PNG/SVG. Full width, it is the
  spine of the piece. Caption underneath.
- 4 pull quotes marked `<!-- PULL QUOTE -->`. The first and last are the two worth
  setting big.
- Section headings are the only separators. There are no horizontal rules, on purpose.
- Links are inline markdown. Keep them live.
- Delete this comment block before publishing.
-->

**Title:** The backend never became public

**Subtitle:** I put my site on a free server. Here is the mental model I was missing, and every error that taught it to me.

Open a terminal, any terminal, and run this:

<!-- IMAGE 1 -->
```
curl raghav56.tech
```
*Caption: try this before you keep reading.*

You get a business card. Coloured, boxed, laid out for a terminal. Open [the same
URL](https://raghav56.tech) in a browser and you get an ordinary website instead. Same
address, same server, two completely different things, decided by one header.

That trick is the last 5% of this story. The other 95% was discovering that a server does
almost nothing you expect, and that every layer between your code and a stranger's browser
fails silently and separately.

I have done this twice now, on two machines, seven months apart. The second time took an
afternoon. The first took weeks, and the difference between them is this newsletter.

## The free machine is real, and it is not a toy

[Oracle Cloud's Always Free tier](https://www.oracle.com/cloud/free/) is not a trial and
does not expire after twelve months. The ARM allocation is **4 CPUs and 24 GB of RAM**,
permanently, for nothing. The box everyone recommends for six dollars a month is 1 CPU and
1 GB.

Here is mine, right now:

<!-- IMAGE 2 -->
```
$ nproc
4

$ free -h
               total        used        free   available
Mem:            23Gi       3.1Gi       686Mi        20Gi

$ uptime -p
up 12 weeks, 6 days
```
*Caption: four cores, 23 GB, three months of uptime, zero rupees.*

That machine runs a website, a Postgres database, a log aggregator, a metrics dashboard,
and two other people's college projects, on 3 GB.

Two things nobody warns you about. **You will see "Out of capacity for shape
VM.Standard.A1.Flex"**, probably several times. That is a queue, not a rejection. Retry at
a different hour, pick a less obvious home region at signup (you cannot change it later),
or create a smaller instance and resize it after, which often works when creating at full
size does not.

And **your machine is ARM**. Download `arm64` builds, not `amd64`. An x86 binary gives you
`Exec format error` and no other hint.

## The mental model I was missing

This is the thing I wish someone had drawn for me on day one. Not the tools. The journey.

<!-- DIAGRAM -->
```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "fontFamily": "ui-monospace, SFMono-Regular, Menlo, monospace",
    "fontSize": "14px",
    "lineColor": "#64748b",
    "primaryTextColor": "#0f172a"
  },
  "flowchart": { "curve": "basis", "nodeSpacing": 45, "rankSpacing": 50 }
}}%%
flowchart TD
    A(["🌐  Browser or curl"]) --> B["DNS<br/>name to public IP"]

    subgraph GATE ["🔒 TWO FIREWALLS · both deny by default"]
        direction TB
        C{"Cloud firewall<br/>Oracle VCN security list<br/>invisible from the server"}
        D{"Host firewall<br/>iptables, policy DROP"}
    end

    B --> C
    C -->|"port not opened"| X1(["⏳ hangs, then times out"])
    C -->|"allowed"| D
    D -->|"dropped"| X1
    D -->|"accepted"| E

    subgraph BOX ["🖥️ YOUR SERVER"]
        direction TB
        E["⚡ Caddy on :80 and :443<br/>the ONLY public process"]
        F{"which Host header?"}
        G["Bun on 127.0.0.1:2056<br/>unreachable from outside"]
        H["static files in /var/www"]
    end

    E --> F
    F -->|"unknown domain"| X2(["🚫 respond 444"])
    F -->|"/api/*"| G
    F -->|"everything else"| H
    G --> I(["✅ Response"])
    H --> I

    classDef gate  fill:#fef3c7,stroke:#b45309,stroke-width:2px,color:#78350f
    classDef fail  fill:#fee2e2,stroke:#b91c1c,stroke-width:2px,color:#7f1d1d
    classDef proxy fill:#4338ca,stroke:#312e81,stroke-width:3px,color:#ffffff
    classDef app   fill:#e0e7ff,stroke:#6366f1,stroke-width:1.5px,color:#312e81
    classDef ok    fill:#dcfce7,stroke:#15803d,stroke-width:2px,color:#14532d
    classDef start fill:#f1f5f9,stroke:#64748b,stroke-width:1.5px,color:#0f172a

    class C,D gate
    class X1,X2 fail
    class E proxy
    class G,H app
    class I ok
    class A,B start

    %% edges 2, 4, 7 are the three ways a request dies. keep these indices in sync
    %% with the edge order above if you ever reorder the arrows.
    linkStyle 2,4,7 stroke:#b91c1c,stroke-width:2.5px,stroke-dasharray:5 4

    style GATE fill:#fffbeb,stroke:#f59e0b,stroke-width:1px,color:#78350f
    style BOX  fill:#f8fafc,stroke:#94a3b8,stroke-width:1px,color:#0f172a
```
*Caption: three gatekeepers before your code ever runs. Any one can say no, and none of them will tell you.*

Notice where your application actually sits. Bottom of the diagram, on `127.0.0.1`, which
means it cannot receive traffic from the internet at all. Exactly one process is exposed,
and it is not yours.

<!-- PULL QUOTE -->
> Your app is not on the internet. Something in front of it is.

## It runs. You close the laptop. It stops.

Your first deploy is you, typing `bun run server.js` into an SSH session. It works. You are
delighted. Then your wifi drops and the site dies with it, because your program was a child
of your login session and the session ended.

The usual first fixes are `nohup` and `tmux`. Both survive logout. Neither restarts your app
when it crashes, survives a reboot, or leaves any trace the service exists.

The real answer was already installed.
[systemd](https://www.freedesktop.org/software/systemd/man/systemd.service.html) started
every other service on your box and will happily start yours. My first attempt, from that
earlier server:

<!-- IMAGE 3 -->
```ini
[Service]
User=ubuntu
WorkingDirectory=~/raghav/Agent_kdg
ExecStart=uv run main.py
StandardOutput=append:logs/server.log
```
*Caption: four separate mistakes in four lines. Can you spot them?*

systemd told me, in language that is genuinely useful once you can read it:

<!-- IMAGE 4 -->
```
Loaded: bad-setting
Main PID: 3476 (code=exited, status=203/EXEC)

WorkingDirectory= path is not absolute: ~/raghav/Agent_kdg
```
*Caption: three error strings worth memorising. You will meet all of them.*

**`path is not absolute: ~/...`** means systemd does not expand `~`. Tilde expansion is a
*shell* feature and there is no shell here. Same for `$HOME`, globs, and `&&`.

**`status=203/EXEC`** means systemd could not execute what you named. Wrong path, missing
file, or missing execute bit.

**`Loaded: bad-setting`** means the unit file is invalid, so nothing ever ran. That differs
from `Loaded: loaded` plus `Active: failed`, which means your program started and exited.
One is your config, the other is your code.

A quieter fourth: relative paths in `StandardOutput=append:logs/server.log` resolve against
`/`, not your working directory.

Then the one that catches everybody exactly once. Both commands succeed. Nothing runs:

<!-- IMAGE 5 -->
```
$ sudo systemctl start agent
$ pgrep -f "uv run main.py"
$            # nothing. no error. no output.
```
*Caption: silence is the worst error message.*

`ExecStart` said `uv`, and systemd has no idea where that is. **systemd does not read your
`.bashrc`.** Neither does cron, nor `ssh host "command"`, which is how your CI will deploy.
Every tool in your home directory is invisible to all three. Run `which uv`, paste the full
path, and it will run for months untouched. Mine did.

## Nothing can reach it

The failure that wastes days, and the diagram above is the whole answer.

**Layer one is what address your app bound to.** A socket binds to an address *and* a port.
`127.0.0.1` means the kernel only delivers packets that came from this same machine. Not
"is blocked from the internet". Cannot receive from it. `0.0.0.0` means every interface,
public one included.

Build the habit of reading the address column, never the port:

<!-- IMAGE 6 -->
```
$ ss -tlnp
LISTEN  127.0.0.1:2056   bun     ← unreachable from outside
LISTEN          *:443    caddy   ← the only public door
```
*Caption: same machine, same kind of app, completely different exposure.*

In Express, `app.listen(PORT)` with no host binds `0.0.0.0`. Every tutorial writes it that
way, and it is the most common accidental exposure in student projects. Pass
`app.listen(PORT, "127.0.0.1")`.

**Layer two is the host firewall.** Every guide says `sudo ufw allow 80`. On the Oracle
image `ufw` is not installed and that command does not exist. Rules are raw iptables with a
default DROP policy, saved with `netfilter-persistent`. And while you are in there: **never
run `iptables -F` on Oracle Cloud.** The default rules carry cloud metadata, DHCP, NTP, and
the iSCSI mount for your boot volume. Flushing them can cost you the root disk.

**Layer three is the cloud firewall**, invisible from inside the machine. Oracle's security
list opens port 22 and nothing else. Everything on the box can be correct and you still get
nothing, because the packet never arrived.

What identifies the layer in one second is the *shape* of the failure, not the message:

- Hangs, then times out → a firewall dropped it silently. Layer two or three.
- Instant `connection refused` → it reached your machine, nothing was listening. Layer one.
- Connects, then nothing → your app is broken. Never a network problem.

One correction to myself. Before you have a proxy, `0.0.0.0` is the *correct* answer and
`127.0.0.1` will drive you mad, because nothing outside can reach it, including you. Once a
proxy exists, everything moves back to loopback. Same question, opposite right answers, so
check which world a tutorial was written for.

## The receptionist

Eventually a friend wants to host their project on your box too, and there is exactly one
port 443.

The way I ended up explaining this out loud to a batchmate is the version that finally
stuck for both of us. **The proxy is a receptionist.** The public IP is the building
entrance, the receptionist sits at the front desk, your backends are departments upstairs.
Nobody walks in off the street and straight into a department.

Which answers what confused me for months: if my backend is on `127.0.0.1`, how is anybody
on the internet using it? They are not. They are talking to the receptionist.

<!-- PULL QUOTE -->
> The backend never became public. That was the whole point.

I started on [nginx](https://nginx.org/en/docs/) and moved to
[Caddy](https://caddyserver.com/docs/). Learn nginx anyway, it is on every server you will
ever inherit. But know the trade. In nginx these four lines are mandatory and nothing warns
you:

<!-- IMAGE 7 -->
```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```
*Caption: forget the last line and your HTTPS site starts emitting http:// links.*

The Caddy equivalent is the entire file, unedited, from my server:

<!-- IMAGE 8 -->
```caddyfile
api.example.com {
    reverse_proxy localhost:5001
}
```
*Caption: obtains a certificate, renews it forever, redirects HTTP to HTTPS, sets all four headers, enables HTTP/2 and HTTP/3.*

## The 403 that redesigned my server

My first attempt at serving files pointed the web server straight at my project folder.
Every request returned `403` with a `content-length` of zero. No message. TLS fine. Routing
fine. The file existed and `ls -l` said it was world readable.

This command solves it, and it is the most useful thing in this newsletter:

<!-- IMAGE 9 -->
```
$ namei -l /home/ubuntu/raghav/site/static/index.txt
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu
drwx------ ubuntu ubuntu raghav          ← 700. there it is.
-rw-rw-r-- ubuntu ubuntu index.txt       ← world readable, and irrelevant
```
*Caption: `namei -l` walks every directory in a path. `ls -l` on the file lies to you.*

**Directory permissions compose.** To read a file you need traverse permission on every
directory above it. Mine being readable meant nothing, because a folder two levels up was
`700`.

The obvious fix is `chmod 755` on your home directory. Do not. That makes every project,
key, and dotfile you own readable by every other account on the machine, to fix one file.

I refused, and that refusal is where my architecture came from. The source tree stays `700`
and the web server genuinely *cannot* read it. The build compiles into a separate folder
and copies the output to `/var/www`, which holds nothing but HTML, CSS and images. If a
file server ever has a path traversal bug, it leaks pages that were already public instead
of my `.env`, my SSH keys, and my `.git` directory.

<!-- PULL QUOTE -->
> The 403 was not a problem to work around. It was the filesystem telling me my design was wrong.

## Certificates are easy. Renewals are where you fail.

[Let's Encrypt](https://letsencrypt.org/) gives you a certificate in one command. Keeping
one is the hard part, and a certificate that stops renewing takes your site down ninety days
after you last thought about it.

On my box, as I write this, **both certificates have broken renewal**, for two different
reasons. One uses a *manual* authenticator, so renewal stops and waits for a human to add a
DNS record. At 3am, nobody answers. The other renews through the nginx plugin, and I moved
to Caddy months ago. **Renewal configs do not follow you when you change web servers.**

A third trap waits even when renewal works: the hooks directory is empty by default, so
nothing reloads your web server afterwards. The certificate on disk is fresh, the one being
served is expired, and every check against the file says everything is fine.

So: run `sudo certbot renew --dry-run` today, which surfaces all three at once. Write the
reload hook. Or, if you do not specifically need a wildcard certificate, use Caddy and skip
this section of your life entirely.

## Check the advice, including mine

Debugging that first server, I was told confidently that `nohup` forks a child process, and
that this explained a PID mismatch I was chasing. It is wrong. `nohup` ignores the hangup
signal, redirects output, then replaces itself with your command in the same process. Same
PID throughout. Ten seconds with `nohup sleep 40 &` and `ps` proves it.

The actual bug was sitting in timestamps I had already pasted: the process I was staring at
had started an hour earlier on a different terminal. A leftover, still holding the port,
while the copy I had just launched died on `EADDRINUSE`.

And a fresh one, from an hour before publishing this. I backed up my Caddy config as
`raghav56.tech.caddy.bak` and left it *inside* the `sites-enabled/` folder. That folder is
imported with a `*` glob, so Caddy loaded the backup as a second site and died on a
duplicate definition. Nothing broke, because the deploy runs `caddy validate` before `caddy
reload`, and validate refused.

<!-- PULL QUOTE -->
> Validate before you reload. It turns a typo into a failed deploy instead of an outage.

## Where to start

The order that works: get the machine, get *something* responding on a port, put a proxy in
front, add the domain, add TLS, then automate the deploy. Each step exists because the
previous one broke.

- [Oracle Cloud Always Free](https://www.oracle.com/cloud/free/) for the server
- [GitHub Student Developer Pack](https://education.github.com/pack) for a free `.tech` domain
- [Caddy](https://caddyserver.com/docs/quick-starts/reverse-proxy) for the proxy and automatic HTTPS
- [Tailscale](https://tailscale.com/) so your dashboards live on a private network, not a public subdomain
- `man ss`, `man namei`, `man systemd.service`. Genuinely.

The full sixteen-part version, with every config and every quirk I collected, is at
[raghav56.tech/blog](https://raghav56.tech/blog).

If you take one thing from this: your app is not on the internet. Something in front of it
is. Once that clicks, the rest is just layers.
