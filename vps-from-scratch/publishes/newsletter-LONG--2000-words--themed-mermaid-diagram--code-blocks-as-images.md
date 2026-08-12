# vps from scratch

## WHAT THIS FILE IS

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
  render it as-is at <https://mermaid.live> and drop in the PNG/SVG. Full width, it is the
  spine of the piece. Caption underneath.
- 4 pull quotes marked `<!-- PULL QUOTE -->`. The first and last are the two worth
  setting big.
- Section headings are the only separators. There are no horizontal rules, on purpose.
- Links are inline markdown. Keep them live.
- Delete this comment block before publishing.

# Article

**Title:** Hosting a web service on a free vps

**Subtitle:** I put my site on a free server. Here is the mental model I was missing, and every error that taught it to me.

Open a terminal, any terminal, and run this:

<!-- IMAGE 1 -->
```bash
curl raghav56.tech
```

*Caption: try this before you keep reading.*

You get a business card. Coloured, boxed, laid out for a terminal. Open [the same URL](https://raghav56.tech) in a browser and you get an ordinary website instead.
Same address, same server, two completely different things, decided by one header.

That trick is the last 5% of this story. The other 95% was getting a free server,
discovering how exactly does a web request works, all layer
between your code and a stranger's requester (browser or terminal),
and every way that can go wrong.

I have done this multiple times now, on two machines, seven months apart.
The first time it took Weeks, there was sooo much that went wrong, and
the second time was still a whole night, you always learn so much.
Ill try to explain the difference between them is this newsletter, so you can do it in a few days.

A few things before we start:

I don't have nearly enough space to write about everything, I'll name drop a bunch and implicitly  leave many things to be done via:

- The loop of asking LLMS "how to deep guide with how and why and what else",
- It works yes, and these questions are must while following a guide,
- Also the how, why are important, otherwise believe me second time might still be in weeks

I'll focus on listing things that AI failed me with, it's perfectly valid to just give this blog to an AI and ask it to guide along if you want that, me personally prefer reading guides and asking my own questions in multiple chats with forks of forks of forks.

I got helped by a bunch of people throughout, seniors, peers, I am really grateful.

## The free machine is real, and it is not a toy

[Oracle Cloud's Always Free tier](https://www.oracle.com/cloud/free/) is not a trial and you get it forever.
The ARM allocation is **2 CPUs and 12 GB of RAM** (used to be double that,
you are already seriously missing out, don't delay).

Github student pack and other trials are temporary and give a fraction of resources of even the current free quota.

Here is an image of me ssh-ing into and resource stats of my machine:

<!-- IMAGE 2 -->
![alt text](image.png)

This machine hosts multiple web services, databases, backends of mine and my friends.

Yes you'll need a credit or debit card with international payments and  
1 SGD for normal account here.

Make your oracle account tenancy in some good region (mine is US Ashburn), Hyderabad is filled that was my first attempt. This will help in next step.

Make sure to set up 2FA and then passkey for the account, it helps a lot.

### The Instance and VCN

#### VCN

This step can technically be skipped, VCN can implicitly be created during the creation of Instance.

Go here for allotting an VCN (Virtual Cloud Network):

![alt text](image-2.png)

This is what connects your Instance to the Internet, connections to and from go through.

Also this is where you'll be managing the subnet, and the security list.

Here's the security list:

![alt text](image-3.png)

Use the Security rules in "Default Security List for [server name]"

This is the foundation you'll need for initial setup, later if you the `tailscale` thing properly, you def wont be dealing with this day-to-day.

#### Instance

Go here for allotting an instance

![alt text](image-1.png)

1. You will see **"Out of capacity for shape VM.Standard.A1.Flex"**, you'll have to probably try several times.

- Remember the tenancy from previous step.
- If it still doesn't work upgrade to a PAYG (Pay as you go) account.

It'll require holding 11-12k Rs for PAYG upgrade, it has been seriously worth it for me,  
ofc these will be refunded and as long as you're smart you'll never be charged.

2. **Your machine is ARM**. Download `arm64` builds, not `amd64`. An x86 binary gives you
`Exec format error` and no other hint.

For the configs ask some llm, just use the best free quota allows, storage, vcpu, etc whatever, its mostly personal preference thing, I used the max though I got 3 accounts so whatever.
For OS I used ubuntu minimal of the LTS version at that time.

You will here be, creating via the interface or providing your own, a rsa based ssh keypair. This is the only easy way to log in, so keep it safe.
More on this in the next section.
\* Not ed25519, rsa.

Note the Public IP of the instance, you'll need it for many things including to ssh into the server.

![alt text](image-4.png)

## SSH

Two things that make things soo convenient if person I'm working with knows are git and ssh.

Be careful about whats to be done on your local machine and what the server.  
By default, for this section of guide, its mostly on your local machine, like things on server also but you'll know what, when you're doing then.

The private key of the pair in previous step was to be saved at: `~/.ssh/`.

The command to ssh into the server is:

```bash
ssh username@server_public_ip

# eg for me
ssh ubuntu@150.xx.xx.xx
# with tailscale
ssh ubuntu@oracle
# with ssh config
ssh oracler
```

It won't work as it is, you'll have to allow your machine to reach and access the server in 3 layers.

1. The first is cloud firewall, which is the security list in VCN from before, it has to allow port 22 for your machines public IP.
Ask an llm how to find it, mp curl something with `-v4` or some system command.

2. The second is the host (your instance) firewall, which is the iptables on the server.
Allow same IP:port as step 1, ask llm for commands.
Here guides and llms will push you towards `ufw`, but it is not installed by default on the oracle image, and setting it us is an hassle tbh.

3. Authentication (Do read about Authentication vs Authorization sometime).
This was already done by creating and saving the key.
You might have to start the daemon or add the key somehow, ask ai on error.

### SSH config

Now lets make it easier, it is the ssh daemon which manage the above connection, they read a config file,  
`~/.ssh/config`, if not already made, create it.  
Now configure it, ask some llm what exactly.

Here's an scaffolding from my setup,

```text
Host oracler
    User ubuntu
    HostName [oracle/Public_IP]
    IdentityFile ~/.ssh/ssh-key-[exact path].key
    IdentitiesOnly yes
    AddKeysToAgent yes
    TCPKeepAlive yes
    ServerAliveInterval 60
    ServerAliveCountMax 10
```

Ask what each thing means to a llm.  
Hostname is interesting, its initially the public IP of the server, but later when we use tailscale, it will be the tailscale hostname, the one alloted by MagicDNS feature, remember to enable (if its not) during next step.

## Tailscale and VPN

Tailscale is a decentralised VPN that makes your server and your local machine part of the same private network, so you can access it without exposing them to the public internet.

[set it up](https://tailscale.com/) on both your server and your local machine.
Basically download and loin with same account and add as a service with systemd, follow the docs or ask ai.
Once set up, you can replace public IP from above step to the tailscale hostname in your ssh config, and ssh into your server without exposing it to the public internet(bypass the need for dealing with the layer 1 defence from above).

Theres a lot one can do with tailscale, explore.

## The mental model I was missing

I wish I could give my past self this diagram.

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

*Caption: three gatekeepers before your code ever runs. Any one can say no, and none of them will tell you. As you saw above*

\* This uses specific things name, for eg there could be systemd or nginx in place of caddy, though above is a good default

Notice where my applications actually are attached, bottom of the diagram, on `127.0.0.1` i.e. localhost, only processes on the machine can access them.  
Read about Host = IP:port, difference bw `0.0.0.0` and `localhost`

> Your app is not on the internet. Something in front of it is.

## It runs, you close the laptop, it stops

Your first task while deploying is making things work locally, on your own laptop or the SSH session.
Then comes persistence, when you close the session or wifi drops, the site dies with it, because your program was a child of your login session and the session ended.

The usual first fixes are `nohup` and `tmux`. Both survive logout. Neither restarts your app
when it crashes, survives a reboot, or leaves any trace the service exists.

The real answer is a system utility: [systemd](https://www.freedesktop.org/software/systemd/man/systemd.service.html).
It started every other service on your box and will happily start yours, spend some time getting familiar with this.
\* ofc there are abstractions over abstraction in our field, ik about  `runc`

Every other program you can use for this purpose is an abstraction over it, pm2, docker, etc.

Learn things like `systemctl status agent`, `journalctl -u agent`, `systemctl enable agent`, `systemctl restart agent`, etc.
They help a bunch in debugging and managing your service, while doing other tasks also.

My very first attempt, from that earlier server, for representational purposes, mine was better ofc:

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
```text
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
```bash
sudo systemctl start agent
pgrep -f "uv run main.py"
# nothing. no error. no output.
```

*Caption: because philosophy of posix is no message on succeed, silence is the worst error message.*

`ExecStart` said `uv`, and systemd has no idea where that is. **systemd does not read your
`.bashrc`.** Neither does cron, nor `ssh host "command"`, which is how your CI will deploy.
Every tool in your home directory is invisible to all three. Run `which uv`, paste the full
path.

## Nothing can reach it, caddy

Here we'll setup a reverse proxy, which acc to the analogy I used with my friend is like a portal master or gatekeeper+receptionist, we'll come back to this analogy later.

Before we install anything next is an important concept to understand.

**Layer one is what address your app bound to.** A socket binds to an address *and* a port.  
`127.0.0.1` or `localhost` means the kernel only delivers packets that came from this same machine. "It is blocked from the internet", only processes on the machine, which obv do not require internet to access it can do so.
`0.0.0.0` means every interface, public ones, which can somehow know its public ip included.

Build the habit of reading the address column, never just the port:

<!-- IMAGE 6 -->
```bash
$ ss -tlnp
LISTEN  127.0.0.1:2056   bun     ← unreachable from outside
LISTEN          *:443    caddy   ← the only public door
```

*Caption: same machine, same kind of app, completely different exposure.*

Since most examples only show `port`, people make very suboptimal choices even when they have a reverse proxy.
It's such a waste to have a portal master when labeled portals exist and anyone can walk in (okay there are locks, but yk can be cracked).  
For eg in Bun, `Bun.serve({ port: PORT })` with no hostname binds `0.0.0.0`.  
Pass `Bun.serve({ port: PORT, hostname: "127.0.0.1" })` instead.  
Ask an AI how to use localhost for your stack in production, not just dev.

**Layer two is the host firewall.** Same `ufw` stuff from before, we don't have it on our machine, ignore that all.
Rules are raw iptables with a default DROP policy, saved with `netfilter-persistent`, ask ai how to open.  
And while you are in there: **never** run `iptables -F` on Oracle Cloud.** The default rules carry cloud metadata, DHCP, NTP, and the iSCSI mount for your boot volume. Flushing them can cost you the root disk.

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
```bash
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

## Certificates are easy. Renewals are where you fail

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
