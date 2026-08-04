# Outline

Working title given to the editor:

> how to deploying webservises from scratch on vps: do everything yourself in a modern
> accessible way with all the quirks

Cleaner options for print:

1. **do it yourself: deploying real web services on your own server** (recommended)
2. everything I broke learning to run a server
3. from free tier to production: a VPS, a domain, and all the quirks

## Who this is for

Second-year students who can write a web app and have never put one on the internet that
stays up. They have probably used Vercel or Netlify. Nothing here assumes Linux
administration knowledge; everything assumes they can read code.

## The spine

The series follows the order the failures actually arrive, not the order a reference
manual would use. Each chapter opens with a symptom.

| # | Chapter | Opens with the failure |
|---|---|---|
| 01 | getting the machine | you want a server and every host wants a card |
| 02 | making the box yours | you are on a bare Ubuntu box and everything is unfamiliar |
| 03 | bun from the start | your toolchain works in your terminal and nowhere else |
| 04 | the naive first deploy | it runs. you close the laptop. it stops |
| 05 | systemd | it crashed at 3am and nothing restarted it |
| 06 | binding and firewalls | it says it is listening and nothing can reach it |
| 07 | nginx | you have two apps and one port 80 |
| 08 | getting a domain | nobody types an IP address |
| 09 | tls | the browser says Not Secure |
| 10 | caddy | your TLS config is longer than your app |
| 11 | subdomains and wildcards | you want five services and you have one domain |
| 12 | oxmgr | the process is running but the site is down |
| 13 | deploy scripts and CI | you deploy by SSHing in and remembering nine commands |
| 14 | secrets and privilege | you pushed the API key |
| 15 | observability | someone says the site was down and you have no idea |
| 16 | the quirks checklist | the appendix people screenshot |

## Word budget

Roughly 1000 to 1600 words per chapter, about 18,000 total. The magazine will not run
that. Plan for it:

**If they run one chapter:** 06 (binding and firewalls). It is the most transferable, the
most commonly misunderstood, and it stands completely alone.

**If they run three:** 01, 06, 16. Get a machine, understand why nothing can reach it,
here is everything else that will bite you.

**If they want a single condensed feature (~2500 words):** cut a version from 01, 04, 06,
09 and 16. That is the arc "get a box, put something on it, understand the network,
get a padlock, avoid the traps" and it is a complete story on its own.

The full series is for the website, where length is free.

## Rules the writing follows

- **Symptom first, tool second.** Never open a chapter by naming the tool.
- **The wrong version first.** This box contains real examples of both, including the
  author's own early work. Use them.
- **Real output only.** Every command was run read-only on the live box and its actual
  output is what appears. No invented terminal transcripts.
- **Name your own work, anonymize everyone else's.** raghav56.tech and its subdomains are
  real and named. Other people's projects appear as "a teammate's API" with paths and
  domains changed. No bcrypt hashes, no tokens, no keys, no real server IP.
- No em dashes (house style, see `SKILLs/em-dash-replacement.md` in the site repo).
- Lowercase chapter titles, matching the existing blog voice.

## What the reader has at the end

A machine that costs nothing, a domain, HTTPS that renews itself, a service that restarts
when it dies and gets restarted when it stops responding, subdomains for anything else
they build, `git push` as the deploy command, and enough of a mental model of binding and
firewalls to debug the next thing themselves.

## Loose ends the series admits to

Being honest about these is what separates it from a tutorial. All are live on the box
right now:

- The wildcard certificate cannot renew unattended (`authenticator = manual`).
- Nothing reloads the proxy after a renewal (`renewal-hooks/` is empty).
- One certificate renews through a web server that is no longer running.
- An admin service exists in the templates and in git, is not deployed, and the wildcard
  block silently serves the main site at its subdomain instead of 404ing.
- Grafana and Loki bind all interfaces and are protected only by the firewall.

Chapter 16 lists them. Chapter 17 does not exist, and the series does not pretend the
system is finished.
