# 09. tls: certbot, caddy, and renewals that do not happen

Your domain works. The browser puts "Not Secure" next to it, and on any form it escalates
to a full interstitial warning.

Certificates are free now. Getting one takes a command. Keeping one is where people
actually fail, and this chapter is mostly about that, because a certificate that stops
renewing takes your site down 90 days after you stopped thinking about it.

## What Let's Encrypt is doing

A certificate authority signs a statement that you control a domain. Let's Encrypt does
this for free, automatically, over a protocol called ACME. Before signing, it makes you
prove control by completing a **challenge**. There are two you will meet.

**HTTP-01.** The CA gives you a random token; you serve it at
`http://yourdomain/.well-known/acme-challenge/<token>`; the CA fetches it. Proves you
control the web server that the domain points at.

- Needs port 80 reachable from the internet.
- Fully automatic.
- **Cannot issue wildcards.** There is no way to serve that file for "every possible
  subdomain", so Let's Encrypt will not do it.

**DNS-01.** The CA gives you a token; you publish it as a TXT record at
`_acme-challenge.yourdomain`; the CA queries DNS. Proves you control the whole zone.

- Needs no open ports, works for a server with no public HTTP at all.
- **The only way to get a wildcard** certificate for `*.yourdomain`.
- Needs API access to your DNS provider to be automatic. Without it, it is a manual
  process, and manual processes do not renew.

That last line is the whole chapter.

## The simple path: certbot

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

It obtains a certificate over HTTP-01, edits your nginx config to use it, and sets up
renewal. Three minutes and you have a padlock.

Verify what you have:

```bash
sudo certbot certificates
```

## The permission trick nobody documents

If your web server does not run as root (and it should not), it cannot read a private key
that root owns. The default is `700 root:root` and the failure is a startup error about
permissions on a file that obviously exists.

On this box:

```
$ sudo ls -ld /etc/letsencrypt/live /etc/letsencrypt/archive
drwxr-x--- 4 root caddy 4096 /etc/letsencrypt/archive
drwxr-x--- 4 root caddy 4096 /etc/letsencrypt/live
```

Group-owned by the web server's user, mode `750`. You must do **both** directories,
because `live/` contains symlinks pointing into `archive/`, and fixing only `live/` gets
you a permission error on a path you never referenced.

```bash
sudo chgrp -R caddy /etc/letsencrypt/live /etc/letsencrypt/archive
sudo chmod -R 750 /etc/letsencrypt/live /etc/letsencrypt/archive
```

## Renewal, and three ways it silently stops

Certificates last 90 days. Renewal is supposed to be automatic. On this box, right now,
**both certificates have broken renewal**, for two different reasons. This is what makes
it a good example rather than an embarrassing one: these are the exact failure modes, live.

### Failure 1: a manual authenticator on an unattended machine

The wildcard certificate's renewal config, `/etc/letsencrypt/renewal/raghav56.tech.conf`:

```ini
[renewalparams]
account = <ACCOUNT_ID>
pref_challs = dns-01,
authenticator = manual
server = https://acme-v02.api.letsencrypt.org/directory
key_type = ecdsa
```

`authenticator = manual` means certbot will stop and ask a human to add a TXT record. On
a server, at 3am, nobody answers. The renewal fails, the certificate expires 90 days after
issue, and the first you hear about it is a browser warning.

Two fixes:

**Use a DNS plugin** if one exists for your provider:

```bash
sudo apt install python3-certbot-dns-cloudflare
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d yourdomain.com -d '*.yourdomain.com'
```

`chmod 600` that credentials file. It holds an API token that can edit your DNS.

**Or write an auth hook** that calls your registrar's API:

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  --manual-auth-hook /usr/local/bin/add-txt.sh \
  --manual-cleanup-hook /usr/local/bin/del-txt.sh \
  -d '*.yourdomain.com'
```

**Or, honestly: ask whether you need a wildcard at all.** If you can list your subdomains,
HTTP-01 handles them individually and needs none of this. Get the wildcard only when
adding a subdomain without touching TLS config is worth this much machinery.

### Failure 2: nothing reloads the server afterwards

```
$ ls /etc/letsencrypt/renewal-hooks/deploy/
(empty)
```

Even if renewal worked, the web server has the old certificate loaded in memory and will
keep serving it until something restarts it. So the certificate on disk is valid, the one
being served is expired, and every diagnostic you run on the file says everything is fine.

One file fixes it:

```bash
# /etc/letsencrypt/renewal-hooks/deploy/reload-webserver.sh
#!/bin/sh
systemctl reload caddy      # or nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-webserver.sh
```

Everything in `deploy/` runs after any successful renewal. Write this on the same day you
get your first certificate.

### Failure 3: the renewal config outlives the server it names

The other certificate on this box:

```ini
[renewalparams]
authenticator = nginx
installer = nginx
```

It renews through the nginx plugin. **nginx has been stopped for weeks** and Caddy owns
port 80. That renewal cannot work, and the certificate expires in seven days.

When you migrate web servers, the certbot renewal configs do not follow you. They keep
pointing at the old one, and they fail silently until a browser tells you.

## Test your renewal, today, not in 89 days

```bash
sudo certbot renew --dry-run
```

This runs the entire renewal against Let's Encrypt's staging environment. It will surface
every one of the failures above. Run it the day you set things up.

Then confirm what is actually driving it:

```bash
systemctl list-timers certbot.timer
```

```
certbot.timer  last: 2026-08-04 13:30  next: 2026-08-05 07:02
```

Note there is also `/etc/cron.d/certbot` on this box, which contains a guard that makes it
a no-op when systemd is running the timer. **Do not "fix" a renewal problem by adding
another cron entry.** Check the timer first. Two renewal mechanisms racing each other is
worse than one broken one.

## The alternative: a server that does it for you

Caddy does ACME internally. Given a site block with a real domain, it obtains a
certificate on first request, renews at roughly two thirds of the lifetime, and reloads
itself. No certbot, no cron, no hooks, no permissions problem, nothing to test.

The whole configuration for a TLS-terminated reverse proxy is:

```caddyfile
api.yourdomain.com {
    reverse_proxy localhost:5001
}
```

That is verbatim from this box (with the domain changed), and that site's certificate is
managed entirely by Caddy. Compare against the nginx block plus certbot invocation plus
renewal config plus deploy hook that achieves the same thing.

If you want a wildcard, Caddy can do DNS-01 too, but it needs a provider-specific build
or module, which is the one place the simplicity runs out.

## What this box actually does, and what you should do

This box runs a hybrid: certbot holds the wildcard via DNS-01, and Caddy is told to use
certbot's files:

```caddyfile
(certbot-wildcard) {
    tls /etc/letsencrypt/live/raghav56.tech/fullchain.pem /etc/letsencrypt/live/raghav56.tech/privkey.pem
}
```

while the other site has no `tls` directive at all and Caddy manages it automatically.

That hybrid exists for a boring historical reason: the wildcard came first, from the
nginx era, and it was never migrated. It is not a design. It is drift, and it is why two
renewals are broken.

**If you are starting today: use Caddy, do not install certbot, and skip this entire
chapter's failure modes.** Reach for certbot only if you specifically need a wildcard and
cannot get Caddy's DNS module working for your provider, and if you do, write the deploy
hook on the same day.

## Two things to add once TLS works

```caddyfile
header Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

HSTS tells browsers to refuse plain HTTP for your domain for a year. Add it once you are
confident HTTPS is stable, because it is hard to undo.

And check your work at `ssllabs.com/ssltest`. Caddy defaults score an A. It is a nice
moment.

Next: the server that made all of this shorter.
