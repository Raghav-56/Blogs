# Research: the nginx that came before

nginx is installed but **inactive and disabled**. The configs are still on disk, which
makes them a perfect "here is what the same thing looks like in nginx" exhibit.

```
$ systemctl is-active nginx ; systemctl is-enabled nginx
inactive
disabled
```

`/var/www/html/index.nginx-debian.html` is also still sitting there, untouched. That is
the stock welcome page, and its presence is the marker of the very first stage.

## /etc/nginx/sites-available/raghav56.tech

```nginx
server {
    listen 80;
    server_name raghav56.tech *.raghav56.tech;
    return 301 https://$host$request_uri;
}

server { 
    listen 443 ssl;
    server_name raghav56.tech *.raghav56.tech;
    
    ssl_certificate /etc/letsencrypt/live/raghav56.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/raghav56.tech/privkey.pem;

    return 302 https://resume.raghav56.tech;
}

server {
    listen 443 ssl;

    server_name dev.raghav56.tech;

    ssl_certificate /etc/letsencrypt/live/raghav56.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/raghav56.tech/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:2056;
        
        proxy_http_version 1.1;
        
	# proxy_set_header Upgrade $http_upgrade;
	# proxy_set_header Connection 'upgrade';
	# proxy_cache_bypass $http_upgrade;
       
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Teaching points:

- Three `server` blocks to express what Caddy expresses in one site block, and the TLS
  certificate paths are repeated in every one of them.
- The four `proxy_set_header` lines are the ones you always need and nobody tells you
  about. Without `Host`, the backend sees `127.0.0.1`. Without `X-Forwarded-Proto`, an
  app that builds absolute URLs emits `http://` links on an HTTPS site. Caddy's
  `reverse_proxy` sets all four for you.
- The commented-out `Upgrade` / `Connection` pair is the WebSocket block. In nginx you
  opt in. In Caddy 2 it is automatic.

## /etc/nginx/sites-available/default

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    server_name _;
    return 444; # '444 Connection Closed Without Response' (Great for security)
}
```

Same idea as `http:// { respond 444 }` in the Caddyfile. Without a default server that
drops unknown Hosts, anyone pointing their own domain at your IP gets your site served
back to them.

## Also on disk

`/etc/nginx/sites-available/api.<project>.live` is a certbot-generated block
(`--nginx` installer) with `# managed by Certbot` markers around the TLS lines. This is
the artifact that explains the broken renewal in `03-tls.md`: certbot wrote an nginx
config and recorded `authenticator = nginx`, then nginx was retired.

`/etc/nginx/nginx.conf` is stock. Worth one line in the chapter:

```nginx
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
```

Ubuntu's default still enables TLS 1.0 and 1.1, both deprecated. Moot while nginx is
stopped, but it is a real example of "defaults are not secure defaults", and a reason to
prefer a server whose TLS defaults are maintained upstream.

`/etc/nginx/conf.d/` is empty.
