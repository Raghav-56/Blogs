# 10. caddy

Count the lines in your nginx setup. Three `server` blocks, certificate paths pasted into
each, four `proxy_set_header` lines you had to look up, a certbot invocation, a renewal
config, a deploy hook you have not written yet.

Now count the lines that are about *your site*, as opposed to about TLS and HTTP
plumbing. It is a small fraction.

## The trade

Here is a complete, working, HTTPS-terminated reverse proxy. This is a real file from this
box, in full:

```caddyfile
api.<project>.live {
    reverse_proxy localhost:5001 {
        header_up Upgrade {http.request.header.Upgrade}
        header_up Connection {http.request.header.Connection}
    }
}
```

That obtains a certificate, renews it forever, redirects HTTP to HTTPS, sets the four
forwarding headers, enables HTTP/2 and HTTP/3, and proxies. The two lines inside the block
are for WebSockets and are arguably unnecessary since Caddy handles upgrades natively.

The trade you are making is real and worth stating: **Caddy does more implicitly.** nginx
does exactly what you wrote and nothing else, which is a virtue when you are debugging
something subtle. Caddy has opinions. For a personal server the opinions are the right
ones and the time saved is enormous.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | \
  sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | \
  sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install caddy
```

Stop nginx first, or Caddy cannot bind port 80:

```bash
sudo systemctl disable --now nginx
```

## The structure worth stealing

Most Caddy tutorials put everything in one `/etc/caddy/Caddyfile`. That file grows, and
then a deploy for one site means editing a file that contains every other site.

This box's entire top-level Caddyfile is:

```caddyfile
# Global options block
{
	log default {
		output file /var/log/caddy/system.log
	}
}

# 1. Unrecognized HTTP Request Guard
http:// {
	respond 444
}

# Import all site-specific configuration blocks
import /etc/caddy/sites-enabled/*
```

Three things, and it never changes again.

The `http:// { respond 444 }` block is the same idea as nginx's `default_server`: any
plain-HTTP request whose Host does not match a configured site is dropped with no
response. Without it, someone pointing their domain at your IP gets your site served for
their name, and your logs fill with vulnerability scanners.

Everything else is one file per site in `/etc/caddy/sites-enabled/`. No
`sites-available` symlink dance, because the source of truth is a git repository. More on
that in chapter 13, but the shape is: the site's Caddy config lives **in the application
repo**, and the deploy script copies it into `sites-enabled/` and reloads. Routing changes
get reviewed, versioned, and reverted like code.

## Snippets, and snippets with arguments

Repetition is the thing that makes nginx configs rot. Caddy has snippets, defined with
parentheses and pulled in with `import`:

```caddyfile
(certbot-wildcard) {
    tls /etc/letsencrypt/live/raghav56.tech/fullchain.pem /etc/letsencrypt/live/raghav56.tech/privkey.pem
}
```

Now every site block is one line away from the right certificate, and moving the
certificate is a one-line change instead of a find-and-replace.

Snippets take arguments, which is where they get genuinely useful:

```caddyfile
(common-headers-logging) {
    header {
        X-Request-ID {http.request.uuid}
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    log {
        output file /var/log/caddy/{args[0]} {
            roll_size {args[1]}
            roll_keep {args[2]}
            roll_keep_for {args[3]}
        }
    }
}
```

Imported per site with its own log file and rotation policy:

```caddyfile
import common-headers-logging "raghav56.log" "5mb" "5" "7d"
import common-headers-logging "dev_raghav56.log" "1.5mb" "2" "3d"
```

Security headers defined once, applied everywhere, and each site gets an appropriately
sized log. The rotation is not theoretical: `/var/log/caddy/` on this box holds live logs
alongside rolled `.gz` archives, and none of them is growing without limit. "My disk
filled up with access logs" is a genuine first-year outage and this prevents it.

## handle, route, and matchers

Two directives that look similar and behave differently.

- **`handle` blocks are mutually exclusive.** First match wins; the rest are skipped.
  Think of it as a switch statement.
- **`route` preserves the order you wrote.** Directives inside execute top to bottom
  rather than in Caddy's normal internal ordering.

Named matchers start with `@`:

```caddyfile
@static {
    path *.css *.js *.woff2 *.ico
}
header @static Cache-Control "public, max-age=86400"
```

A real site block, from this box:

```caddyfile
raghav56.tech, *.raghav56.tech {
    import certbot-wildcard
    import common-headers-logging "raghav56.log" "5mb" "5" "7d"

    encode zstd gzip
    root * /var/www/raghav56.tech

    route {
        redir /resume https://resume.raghav56.tech 302

        handle /api* {
            reverse_proxy 127.0.0.1:2056
        }

        handle /health {
            reverse_proxy 127.0.0.1:2056
        }

        handle {
            file_server
        }
    }
}
```

Static files by default, two path prefixes proxied to a local Bun process, one redirect.
`encode zstd gzip` is compression with content negotiation, in three words.

## The payoff example

This is the trick this site is known for, and it is pure config with zero runtime cost:

```caddyfile
@terminal {
    header_regexp User-Agent (?i)(curl|wget|httpie)
}

handle @terminal {
    try_files {path}/index.txt {path}.txt {path} /index.txt
    file_server
}

handle {
    file_server
}
```

Browsers get `index.html`. `curl`, `wget`, and `httpie` get `index.txt`, a
pre-rendered ANSI page sitting in the same directory. Both are static files produced at
build time; nothing is rendered per request.

```
$ curl raghav56.tech
┌────────────────────────────────────────────────────────┐
│ Raghav Gupta  @Raghav-56                               │
│ AI/ML Researcher · Web Developer · Infrastructure &    │
│ DevOps Engineer                                        │
...
```

`try_files` walks the list and serves the first thing that exists, so a real file
(`resume.pdf`, an image) is still served as itself, and anything with no text version
falls back to the card.

Ordering subtlety worth studying: in the full config, `redir /resume` sits *after* a
terminal-UA handler for the same path. So a browser hitting `/resume` gets redirected, and
`curl` hitting `/resume` gets a text file. Same URL, different behaviour, decided by
matcher order inside a `route`.

## Auth, in three lines

```caddyfile
db.raghav56.tech {
    basic_auth {
        raghav <BCRYPT_HASH>
    }
    reverse_proxy 127.0.0.1:8978
}
```

Generate the hash with:

```bash
caddy hash-password
```

That is the entire authentication story for an internal tool. On this box it fronts a
pgAdmin container, so there are two layers: Caddy's basic auth, then pgAdmin's own login.

**Gotcha, and it is a confusing one.** The bcrypt hash format changed in Caddy 2.7. An
old hash produces:

```
base64-decoding password: illegal base64 data at input byte 8
```

which reads like a corrupted config and is actually "regenerate your password hash". This
box still has that message frozen in `systemctl status caddy` from an old failed reload,
months after the config was fixed. Which leads directly to:

## Validate before you reload

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo caddy reload --config /etc/caddy/Caddyfile
```

Always in that order, and in the deploy script this box uses, always joined by `set -e`
so a failed validation aborts before the reload:

```bash
sudo cp "$src" "$CADDY_DEST"
sudo caddy validate --config /etc/caddy/Caddyfile
sudo caddy reload  --config /etc/caddy/Caddyfile
```

`reload` is graceful: zero downtime, in-flight requests finish. `restart` drops
connections. Use reload.

One quirk: **`caddy validate` fails as a normal user** because it cannot open the log file
in `/var/log/caddy/`. That is not a config error. Run it with sudo.

## The admin API

```
$ ss -tlnp | grep 2019
LISTEN  127.0.0.1:2019   caddy
```

Caddy exposes a JSON API on loopback. Ask it what configuration is actually live, which is
different from what is on disk if a reload failed:

```bash
curl -s localhost:2019/config/ | jq '.apps.http.servers'
```

Being able to diff intended config against running config is genuinely useful during a
confusing outage. It is loopback-only by default, which is correct.

## What you get without asking

- HTTP to HTTPS redirect.
- Certificate issuance and renewal.
- HTTP/2 and HTTP/3 (Caddy on this box is listening on UDP 443 for QUIC right now).
- The four `X-Forwarded-*` and `Host` headers on every proxied request.
- WebSocket upgrades.
- OCSP stapling.
- Structured JSON access logs.

Every one of those is a thing you configured by hand in chapter 07, or forgot to.

Next: five services, one domain.
