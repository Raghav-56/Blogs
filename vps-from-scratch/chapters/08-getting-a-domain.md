# 08. getting a domain

You have a working site at `http://<YOUR_SERVER_IP>`. You cannot put that on a CV, in a
bio, or in a message to a recruiter. Nobody types an IP address.

## Get one free, with the Student Pack

The **GitHub Student Developer Pack** is free with a student email or an enrolment
document, and among a large pile of offers it includes free domains:

- **`.tech` for one year** via get.tech. This is where `raghav56.tech` came from.
- `.me` for one year via Namecheap.
- `.dev`, `.app`, and others rotate through the pack.

Apply at `education.github.com/pack`. Approval usually takes a day or two.

Two things nobody tells you:

**Renewal is not free.** A `.tech` domain runs roughly $40 to $50 a year after the first.
Set a calendar reminder for eleven months out. Decide then whether to renew or move to a
`.com` (about $12/year), and do it before it expires, not after, because the recovery
process is unpleasant and the domain may be sniped.

**Pick a name you will still want in five years.** This is the one durable artifact of the
exercise. `firstnamelastname.tech` or a short handle beats anything clever.

If you cannot get the pack, `.xyz` and `.site` are frequently a few dollars for the first
year, and Cloudflare Registrar sells at wholesale cost with no markup, which makes it the
honest choice for a long-term domain.

## How DNS actually works

Three steps, and understanding them makes every DNS problem debuggable.

1. Your browser asks a resolver: "what is the address for raghav56.tech?"
2. The resolver walks down from the root to `.tech`'s servers, which say "ask these
   nameservers".
3. Those nameservers hold your records and answer with an IP.

Those authoritative nameservers are set at your **registrar**, and they are what you
change if you move DNS to a different provider. On this domain:

```
$ dig NS raghav56.tech +short
tech-domains.earth.orderbox-dns.com.
tech-domains.mercury.orderbox-dns.com.
tech-domains.venus.orderbox-dns.com.
tech-domains.mars.orderbox-dns.com.
```

That is get.tech's own DNS. It is perfectly adequate. Some people move nameservers to
Cloudflare for a nicer interface, faster propagation, and an API. Either is fine; do not
do it on day one while you are also learning everything else.

## The records you need

Only two types matter to start.

**A record**: name to IPv4 address.

| Type | Name | Value |
|---|---|---|
| A | `@` | `<YOUR_SERVER_IP>` |
| A | `*` | `<YOUR_SERVER_IP>` |

`@` is the apex, the bare domain. `*` is a wildcard: it answers for every subdomain that
does not have its own record. That one line is what lets you add `blog.`, `api.`, `dev.`
later with zero DNS work. Chapter 11 is about the consequences, good and bad.

**CNAME record**: name to another name.

| Type | Name | Value |
|---|---|---|
| CNAME | `resume` | `raghav-56.github.io.` |

This box does exactly that:

```
$ dig CNAME resume.raghav56.tech +short
raghav-56.github.io.
```

So `resume.raghav56.tech` is served by GitHub Pages while everything else is served by
the VPS. **One domain can point at as many different hosts as you like.** Static things go
to Pages or Cloudflare Pages for free and stay up when your server does not; dynamic
things live on the VPS. There is no reason to be purist about it.

Two rules on CNAMEs that will save you an afternoon: they cannot coexist with other
records on the same name, and **you cannot CNAME the apex**. `@` must be an A record.
(Some providers offer "ALIAS" or "CNAME flattening" to work around this. get.tech does
not.)

**TTL** is how long resolvers cache the answer. Set it low, 300 seconds, while you are
changing things. Raise it to 3600 once it is stable.

## Verify with dig, not with your browser

Your browser and your operating system both cache DNS aggressively, so refreshing tells
you nothing.

```bash
dig A raghav56.tech +short          # what does DNS say
dig CNAME resume.raghav56.tech +short
dig NS raghav56.tech +short         # who is authoritative
dig @8.8.8.8 raghav56.tech +short   # ask a specific resolver
```

`dig @8.8.8.8` is the useful one. It bypasses your local cache and asks Google's resolver
directly. If that returns the right answer and your browser does not, your problem is
local caching and it will resolve itself.

Propagation is usually minutes, occasionally a couple of hours, and effectively never the
"24 to 48 hours" that registrar help pages threaten. If it has been an hour and
`dig @8.8.8.8` still shows nothing, you got the record wrong.

## Point the proxy at the name

Once DNS resolves, tell your web server the domain exists. In nginx that is `server_name`;
in Caddy it is the site address itself:

```caddyfile
raghav56.tech, *.raghav56.tech {
    root * /var/www/raghav56.tech
    file_server
}
```

Reload, and `http://raghav56.tech` works.

`https://` does not, yet. That is the next chapter.

## Two small things worth doing now

**Set a low TTL before you migrate anything.** If you ever move to a different server,
you will want DNS to switch over in five minutes rather than an hour. Drop the TTL a day
in advance of the move, not during it.

**Do not put your home IP in a public DNS record.** Obvious once stated, easy to do by
accident when testing from a laptop.

Next: the browser says Not Secure.
