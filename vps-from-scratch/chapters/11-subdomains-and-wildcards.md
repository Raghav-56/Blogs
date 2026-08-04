# 11. subdomains and wildcards

You have one domain and five things you want to put online: the site, a staging copy, a
database admin panel, an API, a dashboard.

Subdomains are free. You already own every possible one. The only question is how much
work each new one costs you, and the answer can be "zero" if you set it up correctly once.

## Three layers again

Adding `blog.yourdomain.com` needs three things to agree:

1. **DNS**: something must resolve `blog.yourdomain.com` to your IP.
2. **TLS**: your certificate must cover that name.
3. **Routing**: your web server must have a block matching that hostname.

Do each one individually and every new subdomain is a fifteen-minute chore across three
systems. Do each one with a wildcard and it costs one config block.

## The wildcard setup

**DNS.** One record:

| Type | Name | Value |
|---|---|---|
| A | `*` | `<YOUR_SERVER_IP>` |

Every subdomain that does not have its own record now resolves to your server. No DNS work
ever again.

**TLS.** A wildcard certificate, which as chapter 09 explained requires DNS-01:

```
$ sudo certbot certificates
  Certificate Name: raghav56.tech
    Domains: raghav56.tech *.raghav56.tech
```

**Routing.** One site block covering both:

```caddyfile
raghav56.tech, *.raghav56.tech {
    import certbot-wildcard
    root * /var/www/raghav56.tech
    file_server
}
```

After this, adding a subdomain is one block in one file. Nothing else.

## The trap: a wildcard shadows your 404s

This is the cost, and it is live on this box right now.

A `*.domain` block matches **everything**. Every typo, every subdomain you have not built
yet, every service you took down. They do not 404. They silently serve whatever the
wildcard block serves.

`admin.raghav56.tech` on this box is supposed to be a content-editing service on a local
port. It is written, committed, documented, present in the config templates, and **not
running**. Nothing listens on its port. There is no `admin.` block in the live Caddy
config.

So what happens when you visit `admin.raghav56.tech`? You get the main site. HTTP 200.
Not an error. Not a 404. Just, quietly, the wrong thing.

The same applies to typos: `blgo.raghav56.tech` returns the homepage, so a broken link in
someone's email looks like it works.

Two ways to handle it, and you want at least one:

**Order your blocks specifically-first.** Caddy matches the most specific host first, so an
explicit `admin.raghav56.tech` block always wins over `*.raghav56.tech`. Define every real
subdomain explicitly and let the wildcard be the fallback, not the default.

**Give the wildcard an honest fallback:**

```caddyfile
*.raghav56.tech {
    import certbot-wildcard
    respond "no such service" 404
}
```

Then list your real subdomains explicitly above it. Now an unbuilt subdomain says so.

## Three subdomains, three patterns

This box has three live subdomains, and each demonstrates a different shape.

### dev. : a proxy behind a password

```caddyfile
dev.raghav56.tech {
    import common-headers-logging "dev_raghav56.log" "1.5mb" "2" "3d"

    route {
        @protected path /*
        basic_auth @protected {
            raghav <BCRYPT_HASH>
        }
    }

    reverse_proxy 127.0.0.1:3056
}
```

A staging environment. Basic auth in front so it is not indexed, not scraped, and not
confused with production.

**Note the smaller log rotation**: `1.5mb`, keep 2, for 3 days. Staging traffic is not
worth the disk that production traffic is worth. Small detail, and it is the kind of thing
you only tune after a disk fills.

Honest note: nothing is currently listening on port 3056, because the dev server is
started by hand when needed. So an authenticated request to `dev.` gets a 502. That is
arguably correct behaviour for a staging box, but it is worth knowing that a proxy pointed
at a dead port gives you a 502 and not a useful message.

### db. : an internal tool, defence in depth

```caddyfile
db.raghav56.tech {
    basic_auth {
        raghav <BCRYPT_HASH>
    }

    log {
        output file /var/log/caddy/db.log {
            roll_size 5mb
            roll_keep 5
            roll_keep_for 7d
        }
    }

    encode zstd gzip
    reverse_proxy 127.0.0.1:8978
}
```

Port 8978 is a pgAdmin container. Three layers between the internet and the database:

1. Caddy basic auth.
2. pgAdmin's own login.
3. The container publishes to `127.0.0.1:8978` only, and Postgres itself publishes **no
   host port at all**, so it is reachable only on the compose network.

The compose file matters as much as the Caddy config:

```yaml
ports:
  - "127.0.0.1:8978:80"     # not "8978:80"
```

Without the `127.0.0.1:` prefix, Docker publishes to all interfaces and writes an iptables
rule that bypasses your INPUT chain entirely. See chapter 06. This is how people put
databases on the public internet without meaning to.

Better still, when you can: put admin tools on your tailnet and skip the public subdomain
entirely. Caddy can enforce that too:

```caddyfile
@not_tailnet not remote_ip 100.64.0.0/10
abort @not_tailnet
```

Two lines and the site simply does not respond to anyone outside your VPN. That snippet
sits commented out in this box's config template, waiting.

### A different domain entirely

```caddyfile
api.<project>.live {
    reverse_proxy localhost:5001
}
```

One Caddy instance serves as many **domains** as you like, not just subdomains of one. A
friend's project can point their DNS at your box and get their own certificate and their
own site block, completely independent of yours. No `tls` directive, so Caddy manages that
certificate automatically.

## Adding a new service: the checklist

This box's own docs formalise the pattern. Three steps:

**1. Bind to localhost only.**

```bash
uvicorn app:api --host 127.0.0.1 --port 2057
```

Never a public interface. The proxy is the only process with a public port.

**2. Add a route.** For a small surface, a path prefix on the existing site:

```caddyfile
handle /api/ai* {
    reverse_proxy 127.0.0.1:2057
}
```

For a larger one, its own subdomain block. The docs make a good point here: do not assume
`/api/*` is forever one process. Give each backend its own prefix so routing stays
unambiguous as you add more.

**3. Supervise it.** An entry in your process manager with a health check and log routing,
so it restarts like everything else. Chapter 12.

Then the deploy script picks up the config change and reloads.

## Keep the routing config in the repo

Worth repeating because it is the single best structural decision on this box: the site's
Caddy config lives in the application repository, and `./deploy.sh caddy` copies it to
`/etc/caddy/sites-enabled/` and reloads.

```bash
sudo cp "$src" "$CADDY_DEST"
sudo caddy validate --config /etc/caddy/Caddyfile
sudo caddy reload  --config /etc/caddy/Caddyfile
```

Routing changes get history, review, and `git revert`. A fresh server can be rebuilt from
the repo. And the config that describes how your app is reached lives next to the app.

Next: the process is running and the site is down.
