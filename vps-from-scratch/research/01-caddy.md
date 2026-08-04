# Research: Caddy

Caddy v2.11.4, `/usr/bin/caddy`, active and enabled. It is the only process on this box
holding a public port. Runs as `User=caddy` from the stock package unit at
`/usr/lib/systemd/system/caddy.service`:

```ini
[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
```

`AmbientCapabilities=CAP_NET_BIND_SERVICE` is the answer to "how does a non-root process
bind port 80". Worth quoting in chapter 04.

## The top-level Caddyfile is three directives

`/etc/caddy/Caddyfile`, verbatim:

```caddyfile
# Global options block
{
	# Adjust administrative global settings if needed
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

That is the structural idea worth teaching: the main file never changes. Each site is one
droppable file in `/etc/caddy/sites-enabled/`. There is deliberately no `sites-available`
plus symlink dance here, because the source of truth is the git repo, not `/etc/caddy`.

`http:// { respond 444 }` drops any plain-HTTP request whose Host does not match a
configured site. 444 is the nginx convention for "close without responding" and Caddy
happily emits it.

## Site file: raghav56.tech

`/etc/caddy/sites-enabled/raghav56.tech.caddy`. Deployed by `./deploy.sh caddy`, which
copies it from the repo. Bcrypt hash redacted.

```caddyfile
(certbot-wildcard) {
    tls /etc/letsencrypt/live/raghav56.tech/fullchain.pem /etc/letsencrypt/live/raghav56.tech/privkey.pem
}

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

http://raghav56.tech, http://*.raghav56.tech {
    @terminal_resume {
        header_regexp User-Agent (?i)(curl|wget|httpie)
        path /resume
    }

    handle @terminal_resume {
        root * /var/www/raghav56.tech
        rewrite * /resume.txt
        file_server
    }

    @terminal {
        header_regexp User-Agent (?i)(curl|wget|httpie)
    }

    handle @terminal {
        root * /var/www/raghav56.tech
        try_files {path}/index.txt {path}.txt {path} /index.txt
        file_server
    }

    handle {
        redir https://{host}{uri} 308
    }
}

raghav56.tech, *.raghav56.tech {
    import certbot-wildcard
    import common-headers-logging "raghav56.log" "5mb" "5" "7d"

    encode zstd gzip

    root * /var/www/raghav56.tech

    @static {
        path *.css *.js *.woff2 *.ico
    }
    header @static Cache-Control "public, max-age=86400"

    route {
        # Match terminal users downloading the resume
        @terminal_resume {
            header_regexp User-Agent (?i)(curl|wget|httpie)
            path /resume
        }
        handle @terminal_resume {
            rewrite * /resume.txt
            file_server
        }

        # Named path rules - matched before UA check (for browsers)
        redir /resume https://resume.raghav56.tech 302

        handle /api* {
            reverse_proxy 127.0.0.1:2056
        }

        handle /health {
            reverse_proxy 127.0.0.1:2056
        }

        @terminal {
            header_regexp User-Agent (?i)(curl|wget|httpie)
        }

        # Terminal UA: pre-rendered ANSI page for the route, real files
        # (assets, resume.pdf) as-is, otherwise fall back to the card
        handle @terminal {
            try_files {path}/index.txt {path}.txt {path} /index.txt
            file_server
        }

        # Default: browser site
        handle {
            file_server
        }
    }
}
```

Things to point at in the chapter:

- **Snippets take arguments.** `(common-headers-logging)` is imported four times with
  different log filenames and rotation policies via `{args[0]}` through `{args[3]}`.
- **`handle` vs `route`.** Directives inside `route` execute in written order.
  `handle` blocks are mutually exclusive: first match wins, rest are skipped. The
  `redir /resume` sits *after* the terminal-UA handle so a browser gets the redirect and
  `curl` gets the text file.
- The plain-HTTP block is not just `redir`. It duplicates the terminal handlers so
  `curl http://raghav56.tech` still returns the card instead of a 308 that curl will not
  follow by default.

Verified working, from the box itself:

```
$ curl -k --resolve raghav56.tech:443:127.0.0.1 https://raghav56.tech/ -A curl/8
┌────────────────────────────────────────────────────────┐
│ Raghav Gupta  @Raghav-56                               │
│ AI/ML Researcher · Web Developer · Infrastructure &    │
│ DevOps Engineer                                        │
...
```

## Site file: db.raghav56.tech

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

Fronts a pgAdmin container. Two auth layers: Caddy basic auth, then pgAdmin's own login.
Verified returns 401 without credentials.

## Site file: the teammate API (anonymized)

```caddyfile
api.<project>.live {
    reverse_proxy localhost:5001 {
        header_up Upgrade {http.request.header.Upgrade}
        header_up Connection {http.request.header.Connection}
    }
}
```

No `tls` directive at all, so Caddy obtains and renews this certificate itself. That is
the contrast with the site above, which is pinned to certbot's files. Both strategies on
one box.

## Live route table

Read from the admin API at `127.0.0.1:2019`, which Caddy exposes on loopback by default:

```
SERVER srv0 listen [':443']
  route 0 match [{"host": ["api.<project>.live"]}]
  route 1 match [{"host": ["dev.raghav56.tech"]}]
  route 2 match [{"host": ["db.raghav56.tech"]}]
  route 3 match [{"host": ["raghav56.tech", "*.raghav56.tech"]}]
SERVER srv1 listen [':80']
  route 0 match [{"host": ["raghav56.tech", "*.raghav56.tech"]}]
  route 1 match null
```

## Gotchas found

1. **`systemctl status caddy` shows a stale error.** The `Status:` line still reads
   `base64-decoding password: illegal base64 data at input byte 8` from an old failed
   reload. The bcrypt hash format changed in Caddy 2.7; a pre-2.7 hash produces exactly
   this confusing message. The current on-disk config is valid: `caddy adapt` returns
   clean and the live config matches disk.
2. **`caddy validate` fails as a normal user** because it cannot open
   `/var/log/caddy/system.log`. Not a config error. Run it with sudo.
3. Caddy still holds its own ACME-issued certificates for both names in
   `/var/lib/caddy/.local/share/caddy/`, left over from before the `tls` directive pinned
   the main site to certbot's files. Harmless, but confusing when you go looking.
4. Log files are `0600 caddy:caddy`, which matters for the log shipper (see
   `05-network.md` and the observability chapter).

## Log rotation is doing its job

`/var/log/caddy/` contains `raghav56.log` (3.5 MB), `db.log` (3.0 MB),
`dev_raghav56.log`, `api_<project>.log`, `system.log`, plus rolled `.gz` archives. The
`roll_size` / `roll_keep` arguments in the snippet are visibly working, which is worth
showing because "my disk filled up with access logs" is a real first-year failure.
