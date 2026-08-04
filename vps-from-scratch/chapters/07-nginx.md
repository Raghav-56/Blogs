# 07. nginx

You have your site running. Now a friend wants to host their project on your box too.

Both need port 443. There is one port 443.

## What a reverse proxy is

A single program that owns ports 80 and 443, looks at the `Host` header of each incoming
request, and forwards it to whichever local service should handle it.

```
                          ┌─ Host: raghav56.tech    → 127.0.0.1:2056
internet → :443 → proxy ──┼─ Host: db.raghav56.tech → 127.0.0.1:8978
                          └─ Host: api.other.live   → 127.0.0.1:5001
```

Your apps move to high ports on loopback. The proxy is the only thing exposed. This
solves the port conflict, and four other problems you have not hit yet:

- **TLS in one place.** One certificate, one renewal, one config. Not per-app TLS in five
  frameworks with five different bugs.
- **Your app stops facing the internet.** Malformed requests, slowloris, header
  smuggling: handled by software written for it, not by your Express app.
- **Static files served properly.** Compression, cache headers, range requests, all for
  free, and faster than your runtime doing it.
- **Restarts stop being outages.** The proxy holds the socket; a restarting backend
  produces a few 502s instead of a dead port.

This chapter is nginx. Chapter 10 replaces it with Caddy. Both are on this box, and nginx
is off. You should learn nginx anyway, because it is what you will find on every server
you inherit for the rest of your life.

## Install

```bash
sudo apt install nginx
```

It starts immediately and serves a welcome page. If you can see it at
`http://<YOUR_SERVER_IP>`, all three layers from chapter 06 are working. That page is a
genuinely useful first test.

Fun fact: `/var/www/html/index.nginx-debian.html` is still sitting on this box, months
after nginx was retired. It is the archaeological marker of stage one.

## The config layout

```
/etc/nginx/
├── nginx.conf              # global; mostly leave alone
├── sites-available/        # all site configs
├── sites-enabled/          # symlinks to the active ones
└── conf.d/                 # extra global snippets
```

You write a file in `sites-available/`, then symlink it into `sites-enabled/`:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default      # remove the welcome page
```

The two-directory dance exists so you can disable a site without deleting its config.

## A real proxy config

This is from this box, the version that ran before Caddy:

```nginx
server {
    listen 80;
    server_name raghav56.tech *.raghav56.tech;
    return 301 https://$host$request_uri;
}

server { 
    listen 443 ssl;
    server_name dev.raghav56.tech;
    
    ssl_certificate /etc/letsencrypt/live/raghav56.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/raghav56.tech/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:2056;
        
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**`server_name` is how routing happens.** nginx compares it against the `Host` header. The
first block catches plain HTTP for the domain and redirects to HTTPS. The second handles
one subdomain.

**Those four `proxy_set_header` lines are not optional**, and nobody warns you about them.
Without them:

- No `Host`: your app sees `127.0.0.1` as the hostname. Any absolute URL it builds is
  wrong, and virtual-host logic inside the app breaks.
- No `X-Forwarded-For`: every request in your logs comes from `127.0.0.1`. Rate limiting
  by IP now rate limits the proxy.
- No `X-Forwarded-Proto`: your app thinks the request was plain HTTP, because the
  connection from the proxy to the app is. It emits `http://` links on an HTTPS page and
  browsers block them as mixed content. This one produces the most baffling bug reports.

Memorise the block, or copy it every time. Caddy sets all four for you, which is a large
part of why chapter 10 exists.

**`proxy_http_version 1.1`** because nginx defaults to HTTP/1.0 upstream, which breaks
keepalive and chunked responses.

## WebSockets are opt-in

The original of that config has these lines commented out:

```nginx
# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection 'upgrade';
# proxy_cache_bypass $http_upgrade;
```

If your app uses WebSockets (a chat, live reload, anything realtime) and they silently
fail to connect through the proxy while working fine on localhost, this is why. nginx will
not forward the protocol upgrade unless you tell it to.

## Reject unknown hosts

Anyone can point their domain's DNS at your IP address. If the proxy has no default
server, whatever site is listed first answers for it, and now your site is serving someone
else's domain.

The default file on this box:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    server_name _;
    return 444; # '444 Connection Closed Without Response' (Great for security)
}
```

444 is an nginx invention: close the connection with no response at all. Do this on every
server you set up. It also cuts down the noise in your logs from bots scanning IP ranges
for `/wp-login.php`.

## Never reload without testing

```bash
sudo nginx -t && sudo systemctl reload nginx
```

`nginx -t` parses the config and reports errors with line numbers. The `&&` means a
failed test aborts before the reload.

**`reload` is not `restart`.** Reload re-reads the config and gracefully hands
connections to new workers, with zero dropped requests. Restart kills the process and
starts a new one, dropping every in-flight connection. Use reload for config changes.

Get this habit now, because the same pattern appears in chapter 10 as
`caddy validate && caddy reload`, and it is the difference between a config typo being a
non-event and being an outage.

## Where nginx will annoy you

Working through the same setup twice, once in each server, the friction points are
consistent:

1. **TLS is entirely manual.** You install certbot, you run it, you hope the renewal
   works, you add the certificate paths to every server block by hand. Chapter 09.
2. **Repetition.** Three `server` blocks for one domain, with the certificate paths
   copy-pasted into each. Get one wrong and that subdomain serves a certificate for
   another name.
3. **Modern protocols cost effort.** HTTP/2 needs a flag. HTTP/3 needs a build with a
   different TLS library. In Caddy both are on by default.
4. **Ubuntu's defaults are aged.** This box's stock `nginx.conf` still has:

   ```nginx
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
   ```

   TLS 1.0 and 1.1 have been deprecated for years. Nobody edits this file. It is a small
   thing, and it is representative: nginx gives you every knob and assumes you will turn
   them, and you will not.

None of this makes nginx a bad choice. It is the most battle-tested web server in
existence, it is on every machine you will ever inherit, and its documentation is
excellent. Learn it.

Then, for your own projects, consider the one that does the tedious parts for you.

Next: nobody types an IP address.
