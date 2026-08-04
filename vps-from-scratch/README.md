# deploying web services from scratch on a VPS

A sixteen-part series on putting real things on the internet yourself: a free server, a
domain, HTTPS that renews, services that restart themselves, `git push` to deploy, and
every quirk that cost me an evening.

Written for first and second years who can build a web app and have never run one that
stays up.

## Why this exists

Everything here is running. The configs are copied off a live Oracle Cloud free-tier box
that serves `raghav56.tech`, a Postgres instance, a metrics stack, and a couple of other
people's projects. Every command was run read-only on that machine and the output you see
is the output it gave.

That includes the parts that are wrong. The box contains my first attempt (a Node process
under systemd trying to bind port 80), a service that is working and badly configured, and
a certificate that is going to stop renewing. The series says so, because the failures are
the useful part.

## Chapters

| # | Title | What it fixes |
|---|---|---|
| [01](chapters/01-getting-the-machine.md) | getting the machine | you want a server and every host wants a card |
| [02](chapters/02-making-the-box-yours.md) | making the box yours | bare Ubuntu is a hostile place to work |
| [03](chapters/03-bun-from-the-start.md) | bun from the start | your toolchain works in your terminal and nowhere else |
| [04](chapters/04-the-naive-first-deploy.md) | the naive first deploy | it runs, you close the laptop, it stops |
| [05](chapters/05-systemd.md) | systemd | it crashed at 3am and nothing restarted it |
| [06](chapters/06-binding-and-firewalls.md) | binding and firewalls | it says it is listening and nothing can reach it |
| [07](chapters/07-nginx.md) | nginx | two apps, one port 443 |
| [08](chapters/08-getting-a-domain.md) | getting a domain | nobody types an IP address |
| [09](chapters/09-tls-certbot-vs-caddy.md) | tls | the browser says Not Secure, and renewals that quietly stop |
| [10](chapters/10-caddy.md) | caddy | your TLS config is longer than your app |
| [11](chapters/11-subdomains-and-wildcards.md) | subdomains and wildcards | five services, one domain |
| [12](chapters/12-oxmgr.md) | the process is running and the site is down | systemd says green, the site returns 502 |
| [13](chapters/13-deploy-scripts-and-ci.md) | deploy scripts and CI | you deploy by remembering nine commands |
| [14](chapters/14-secrets-and-privilege.md) | secrets and privilege | you pushed the API key |
| [15](chapters/15-observability.md) | observability | "was your site down yesterday?" |
| [16](chapters/16-the-quirks-checklist.md) | the quirks checklist | everything above, one line each |

## Where to start

**Read one chapter?** [06, binding and firewalls](chapters/06-binding-and-firewalls.md).
It is the one that explains why your app is unreachable, it applies to every host and
every language, and it stands completely alone.

**Read three?** 01, 06, 16. Get a machine, understand the network, then keep the checklist
open while you build.

**Actually following along?** Start at 01 and go in order. Each chapter opens with the
failure that motivates the next tool, which is the order these problems actually arrive in.

## What you have at the end

A machine that costs nothing. A domain. HTTPS that renews without you. A service that
restarts when it crashes and gets restarted when it stops responding. Subdomains for
anything else you build, at zero marginal cost. `git push` as the deploy command. And
enough of a model of binding, firewalls, and proxies to debug the next thing yourself.

## Repository layout

```
chapters/      the series, in order, plain markdown
research/      the harvested source material every chapter is built on:
               verbatim configs, exact command output, file paths
Chat_exports/  old ChatGPT logs from Nov 2025 to Aug 2026, kept as primary sources
OUTLINE.md     narrative spine, word budget, magazine cuts
```

`research/` is the fact-checked source of truth. Nothing in a chapter asserts something
that is not in there. If you want the raw configs rather than the prose, read that folder.

`Chat_exports/` is where the error messages came from. The live box shows you what works;
those logs show what failed first, with timestamps, including a whole earlier server from
seven months before this one. `research/11-chat-archaeology.md` is the extract. They also
contain at least one credential I pasted while debugging, which is its own lesson and is
noted there.

## A note on what is redacted

All password hashes, API keys, tokens, ACME account IDs, and the server's real IP are
removed. My own work is named directly. Other people's projects on the same box appear
anonymized, with paths and domains changed but their configuration intact, because their
mistakes are instructive and their names are not mine to publish.
