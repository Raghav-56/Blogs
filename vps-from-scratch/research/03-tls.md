# Research: TLS

Two certificate-management strategies coexist on this box. That is the whole lesson of
the chapter, and it happened by accident rather than design, which makes it honest.

## Current certificates

```
$ sudo certbot certificates
  Certificate Name: api.<project>.live
    Key Type: ECDSA
    Domains: api.<project>.live
    Expiry Date: 2026-08-12 02:37:51+00:00 (VALID: 7 days)
  Certificate Name: raghav56.tech
    Key Type: ECDSA
    Domains: raghav56.tech *.raghav56.tech
    Expiry Date: 2026-09-26 16:17:04+00:00 (VALID: 53 days)
```

## The wildcard: DNS-01, manual authenticator

`/etc/letsencrypt/renewal/raghav56.tech.conf`:

```ini
version = 2.9.0
archive_dir = /etc/letsencrypt/archive/raghav56.tech
cert = /etc/letsencrypt/live/raghav56.tech/cert.pem
privkey = /etc/letsencrypt/live/raghav56.tech/privkey.pem
chain = /etc/letsencrypt/live/raghav56.tech/chain.pem
fullchain = /etc/letsencrypt/live/raghav56.tech/fullchain.pem

[renewalparams]
account = <ACCOUNT_ID>
pref_challs = dns-01,
authenticator = manual
server = https://acme-v02.api.letsencrypt.org/directory
key_type = ecdsa
```

Caddy is then told to use those files rather than fetch its own:

```caddyfile
(certbot-wildcard) {
    tls /etc/letsencrypt/live/raghav56.tech/fullchain.pem /etc/letsencrypt/live/raghav56.tech/privkey.pem
}
```

**Why DNS-01 at all:** HTTP-01 proves control of one hostname by serving a file at
`http://<host>/.well-known/acme-challenge/...`. There is no way to serve that file for
"every possible subdomain", so Let's Encrypt will not issue a wildcard over HTTP-01.
DNS-01 proves control of the whole zone by publishing a `_acme-challenge` TXT record,
which is why it is the only path to `*.raghav56.tech`.

## The other cert: HTTP-01 via the nginx plugin

`/etc/letsencrypt/renewal/api.<project>.live.conf`:

```ini
[renewalparams]
account = <ACCOUNT_ID>
authenticator = nginx
installer = nginx
server = https://acme-v02.api.letsencrypt.org/directory
key_type = ecdsa
```

## Three real gotchas, all verifiable on disk

1. **`authenticator = manual` cannot renew unattended.** There is no credentials file
   anywhere (`/root/.secrets`, `/home/ubuntu/.secrets`, `/etc/letsencrypt/*.ini` all
   absent) and no `--manual-auth-hook`. `certbot renew` will stop and ask a human to add
   a TXT record. On a server, nobody is there to answer. The fix is a DNS plugin for your
   provider, or an auth hook script that talks to the registrar's API.

2. **`/etc/letsencrypt/renewal-hooks/` is completely empty.** Even if renewal succeeded,
   nothing would reload Caddy, so the proxy would keep serving the old certificate from
   memory until something restarted it. A one-line `deploy` hook fixes this:

   ```bash
   # /etc/letsencrypt/renewal-hooks/deploy/reload-caddy.sh
   #!/bin/sh
   systemctl reload caddy
   ```

3. **A renewal authenticator can outlive the server it names.** The second cert renews
   via the `nginx` plugin, but nginx is stopped and Caddy owns port 80. That renewal will
   fail. When you migrate proxies, the certbot renewal configs do not follow you.

## What drives renewal

```
$ systemctl list-timers certbot.timer
certbot.timer  last: 2026-08-04 13:30  next: 2026-08-05 07:02
```

```ini
# /usr/lib/systemd/system/certbot.service
ExecStart=/usr/bin/certbot -q renew --no-random-sleep-on-renew
```

`/etc/cron.d/certbot` also exists but contains its own guard that makes it a no-op when
systemd is running the timer. Do not "fix" a renewal problem by adding a second cron
entry; check the timer first with `systemctl list-timers`.

## The permission trick that makes it work

```
$ sudo ls -ld /etc/letsencrypt/live /etc/letsencrypt/archive
drwxr-x--- 4 root caddy 4096 Jun 28 17:15 /etc/letsencrypt/archive
drwxr-x--- 4 root caddy 4096 Jun 28 17:15 /etc/letsencrypt/live
```

Both directories are group-owned by `caddy` and mode `750`. This is how a proxy that runs
as an unprivileged user reads a private key that root owns. The default is `700 root:root`,
which is why "certbot says the cert exists but my proxy says permission denied" is such a
common first-time failure. Note you have to chgrp **both** directories, because `live/`
holds symlinks into `archive/`.

## Caddy's own store

`/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/`
holds live certificates for both names, plus a staging account directory. So Caddy did
successfully obtain its own wildcard at some point before the `tls` directive pinned it to
certbot's copies. Only `certificates.load_files` for the certbot pair appears in the live
TLS config, so certbot's cert is the one actually served.

Practical takeaway for the chapter: **if you are starting fresh and do not need a
wildcard, delete certbot entirely and let Caddy do it.** Caddy's automatic HTTPS obtains,
renews, and reloads with zero configuration and zero cron. The hybrid on this box exists
because the wildcard came first, and it is worth keeping only if you genuinely need
`*.domain`.
