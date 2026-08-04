# 16. the quirks checklist

Everything in the series, one line each, plus the small ones that never earned a section.
This is the page to keep open.

## Getting the machine

- Oracle's free ARM tier is 4 OCPU and 24 GB, permanently. It is not a trial.
- "Out of capacity for shape VM.Standard.A1.Flex" is transient, not a rejection. Retry,
  try a different time of day, or create smaller and scale up.
- Your home region is chosen at signup and **cannot be changed**. Pick a less popular one.
- Paste your SSH public key into the creation form. There is no password login and the
  recovery path is unpleasant.
- Your machine is `aarch64`. Download `arm64` binaries, check for `arm64` Docker images.
  An x86 binary gives you `Exec format error`.
- There is **no swap** by default. On a small shape, add a swap file.
- `free -h` shows a tiny "free" and a large "available". Read the **available** column.
  Linux did not eat your RAM.

## Shell and SSH

- `~/.bash_profile` existing means **`~/.profile` is never read** for login shells.
- **systemd, cron, and `ssh host "cmd"` read none of your shell config.** Use absolute
  paths in anything a machine runs.
- If port 22 is blocked on your network, GitHub answers SSH on 443:
  `HostName ssh.github.com` / `Port 443` in `~/.ssh/config`.
- Commit with your `@users.noreply.github.com` address, not your real one. It is
  permanent and public otherwise.
- The empty `helper =` line in gitconfig is deliberate. It clears inherited credential
  helpers.
- `ServerAliveInterval 60` in your SSH config stops sessions dying on a wifi hiccup.
- OSC 52 (`clip()`) copies from the server to your laptop's clipboard over plain SSH. Your
  terminal may need it enabled.
- Installers that append to `.bashrc` should use delimiter comments. Read your own
  `.bashrc` occasionally and delete what has rotted.

## Runtime

- Bun does **not** reliably auto-load `.env` under a process manager. Declare variables in
  the service config.
- **`HOSTNAME` is already a system variable**, set to your machine's hostname. An app that
  reads it for a bind address will try to bind to `oracler`. Same for `USER`, `HOME`,
  `PATH`, `LANG`.
- Ship a `/health` endpoint on day one. Three separate systems will want it.
- `bun --bun astro build` forces the whole toolchain onto Bun; without `--bun` tools shell
  out to node.
- Do not symlink your runtime into `/usr/local/bin` to fix PATH. You are hiding the
  dependency, not declaring it.

## Ports and binding

- **`ip addr show` never shows your public IP.** Oracle gives the OS a private `10.x`
  address and NATs the public one at the gateway. You cannot bind the public address.
  `curl ifconfig.me` to see it.
- **Bind `0.0.0.0` until you have a reverse proxy, `127.0.0.1` after.** Both are correct
  for their stage. A tutorial telling you one or the other is written for one of those two
  worlds.
- **`app.listen(PORT)` with no host binds `0.0.0.0`**, every interface, the public
  internet included. Pass `"127.0.0.1"`.
- Read the **Local Address** column of `ss -tlnp`, not the port.
- Ports below 1024 need root or `AmbientCapabilities=CAP_NET_BIND_SERVICE`. Never `sudo`
  your app to get port 80.
- **`ufw` is not installed** on the Oracle image. Rules are raw iptables via
  `netfilter-persistent`.
- **Never run `iptables -F` on Oracle Cloud.** The `InstanceServices` chain carries
  metadata, DHCP, NTP, and iSCSI for your boot volume.
- `iptables -A` appends *after* existing rules including DROPs. Use `-I INPUT <n>` and
  check with `--line-numbers`.
- `netfilter-persistent save` or your rules vanish on reboot.
- Oracle's **VCN security list is invisible from the server** and blocks by default. If
  everything on the box is right and it still hangs, it is the console.
- Hang means a firewall dropped it. Instant "connection refused" means nothing is
  listening. Learn the difference; it identifies the layer immediately.
- **`docker run -p 8080:80` bypasses your INPUT rules**, because the DNAT happens in
  PREROUTING. Always write `-p 127.0.0.1:8080:80`.
- Publish no ports at all for databases. Container network only.
- `ss -tlnp` needs sudo to show process names.
- Read `ss -tlnp` monthly and ask what each line is for. `rpcbind` on `0.0.0.0:111` is on
  this image and used by nothing.

## Reverse proxy

- Four `proxy_set_header` lines in nginx are mandatory: `Host`, `X-Real-IP`,
  `X-Forwarded-For`, `X-Forwarded-Proto`. Missing the last one gives you mixed-content
  bugs that make no sense.
- nginx does not forward WebSocket upgrades unless you add `Upgrade` and `Connection`
  headers. Caddy does it automatically.
- `proxy_http_version 1.1`, because nginx defaults to 1.0 upstream.
- Add a **default server that rejects unknown Hosts** (`return 444` in nginx,
  `http:// { respond 444 }` in Caddy). Otherwise anyone pointing DNS at your IP gets your
  site.
- **Always `nginx -t` / `caddy validate` before reload.** With `set -e` in a script, a bad
  config becomes a failed deploy instead of an outage. It earns its keep: I once saved a
  backup as `mysite.caddy.bak` *inside* `sites-enabled/`, and since the import is a bare
  `*` glob it loaded the backup too and died on a duplicate snippet. Validate refused, the
  reload never ran, the live site never noticed.
- **`import dir/*` imports every file in that directory**, whatever the extension. Keep
  backups, drafts, and `.bak` files somewhere else entirely.
- `reload` is graceful, `restart` drops connections. Use reload for config changes.
- `caddy validate` fails as a non-root user because it cannot open the log file. Not a
  config error.
- Caddy's bcrypt hash format changed in 2.7. An old hash gives
  `base64-decoding password: illegal base64 data at input byte 8`, which means "regenerate
  the hash".
- Ubuntu's stock `nginx.conf` still enables TLS 1.0 and 1.1. Nobody edits that file.
- Caddy `handle` blocks are mutually exclusive, first match wins. `route` preserves the
  order you wrote. Matcher order inside a `route` is how you serve different content to
  different clients on the same URL.
- Caddy snippets take arguments (`{args[0]}`). Define headers and log rotation once.
- The admin API on `127.0.0.1:2019` tells you what config is actually live, which differs
  from disk when a reload failed.
- **Set log rotation per site.** `roll_size`, `roll_keep`, `roll_keep_for`. Access logs
  filling a disk takes down everything at once.

## DNS and domains

- GitHub Student Pack gives a free `.tech` for a year. **Renewal is $40 to $50.** Set a
  calendar reminder for month eleven.
- One `A *` record covers every subdomain forever.
- **You cannot CNAME the apex.** `@` must be an A record.
- A CNAME cannot coexist with other records on the same name.
- One domain can point at many hosts. Static things on GitHub Pages, dynamic on the VPS.
- Verify with `dig @8.8.8.8 name +short`, not by refreshing your browser. Both your OS and
  browser cache aggressively.
- Lower the TTL a day *before* you migrate, not during.

## TLS

- HTTP-01 needs port 80 open and **cannot issue wildcards**. Wildcards require DNS-01.
- **`authenticator = manual` cannot renew unattended.** It waits for a human who is not
  there. Use a DNS plugin or an auth hook.
- `/etc/letsencrypt/renewal-hooks/deploy/` is empty by default, so **nothing reloads your
  web server after renewal**. The cert on disk is valid and the one being served is
  expired. Write the hook on day one.
- Renewal configs **do not follow you when you change web servers**. A cert with
  `authenticator = nginx` silently stops renewing once nginx is off.
- `chgrp` **both** `/etc/letsencrypt/live` and `/etc/letsencrypt/archive`. `live/` is only
  symlinks into `archive/`.
- Run `sudo certbot renew --dry-run` **today**, not in 89 days.
- Check `systemctl list-timers` before adding a cron entry. Two renewal mechanisms racing
  is worse than one broken one.
- If you do not need a wildcard, let Caddy handle TLS and never install certbot.

## Subdomains

- **A `*.domain` block shadows every 404.** Typos and unbuilt subdomains silently serve
  your main site with a 200. Add an explicit fallback or list real subdomains first.
- Give staging a smaller log rotation than production.
- A proxy pointed at a port with nothing listening returns 502, not a useful message.
- `@not_tailnet not remote_ip 100.64.0.0/10` plus `abort` restricts a site to your VPN in
  two lines.

## systemd

- **`path is not absolute: ~/...`**: systemd does not expand `~`, `$HOME`, globs, or
  `&&`. There is no shell.
- **`status=203/EXEC`** means systemd could not execute what you named. Wrong path,
  missing file, or missing execute bit.
- **`Loaded: bad-setting`** means the unit file is invalid and nothing ever ran. Different
  from `Loaded: loaded` + `Active: failed`, which means your program exited.
- Relative paths in `StandardOutput=append:logs/app.log` resolve against `/`, not
  `WorkingDirectory`.
- `start` and `enable` can both succeed with nothing running, if `ExecStart` names a
  binary systemd cannot find. Check `pgrep`, not the exit code.
- `systemctl enable` just creates a symlink into `<target>.wants/`. That is the whole
  mechanism.
- Read `Main PID` in `systemctl status`. If it names `npm`, `uv`, `sh`, or `poetry` rather
  than your program, systemd is supervising a launcher.
- `daemon-reload` after **every** unit file edit, or you are running the old file.
- `enable` and `start` are different. `start` alone dies at reboot; `enable` alone does
  nothing until reboot. Use `enable --now`.
- `After=network.target` means the stack started, not that an IP exists. For that you need
  `After=network-online.target` **and** `Wants=network-online.target`.
- **Never `ExecStart=npm run dev`.** systemd ends up supervising npm, three forks from the
  real process, and every restart guarantee in the file becomes void.
- `ExecStart` takes no shell: no pipes, no `&&`, no variable expansion, no globs.
- `User=` must match who owns `WorkingDirectory`, or git reports "dubious ownership".
- User units need `sudo loginctl enable-linger <user>` or they die when you log out.
- `systemctl cat myapp` shows what systemd actually loaded. Reach for it when an edit
  seems to have no effect.
- systemd knows your process **exists**. It has no idea whether it **works**.

## Deploys

- **Pin GitHub Actions to commit SHAs, not tags.** A tag can be repointed at code that
  runs with your secrets.
- Set `permissions: contents: read` explicitly.
- Non-interactive SSH has no PATH. Export it as the first line of any remote script.
- `envs: GH_PAT` in ssh-action: variables must be explicitly forwarded to the remote
  shell.
- **Guard against a dirty working tree before `git pull`**, or a forgotten server-side
  hotfix gets clobbered or halts the deploy mid-flight.
- `set -euo pipefail` on line 2 of every script. Without `-e`, a failed build deploys the
  previous build.
- A static site generator wipes its output directory. Generate everything else *after* it.
- `rsync -a --delete` or deleted pages stay live forever.
- **A child script cannot `export` back to its parent.** Write `KEY=value` to a temp file
  and `source` it.
- Build your JSON with `jq -n --arg`. Hand-built JSON breaks on the first quote character
  in a commit message.
- `if: always()` on your notification step, or you get told about every success and no
  failure.
- Health checks should **exercise the feature**, not just the port.
- Retry health checks (10 attempts, 5 seconds) so a check right after a reload does not
  fail on a race.
- Keep CI-specific code in a thin shim; keep the real logic in a script that runs on your
  laptop.
- If you `rsync --delete` into the live web root, a broken build is an outage. Know that
  you chose that.

## Secrets

- **A 403 with `content-length: 0` from a file server is almost always a directory
  permission**, not a missing file. Run `namei -l /full/path/to/file`.
- **Directory permissions compose.** You need traverse on every directory above a file.
  `ls -l` on the file itself tells you nothing. The fix is a public web root, never
  `chmod 755 ~`.
- **Gitignored is not safe.** Mode `664` on a multi-user box means everyone can read your
  key. `chmod 600`.
- **Never paste a config containing a password hash into a chat, an issue, or a gist.**
  You will export that chat later and commit it. Ask me how I know.
- Commit `*.template` and `.env.example` beside every gitignored config, so the shape is
  documented.
- Document what happens when an optional key is **absent**, so the project runs without
  every credential.
- **Never serve a directory containing your source.** Build elsewhere and serve that. An
  exposed `.git` directory is your entire history including deleted secrets.
- Source tree `700`, web root `755`, and the web server user cannot read the source.
- If you leak a key: **rotate first**, immediately. Cleaning git history is tidying up,
  not remediation.
- Turn on GitHub push protection. Add a `pre-commit` hook that refuses `.env`.
- Be careful with `set -x` in scripts that touch secrets.
- A secret referenced by nothing is worse than no secret, because you stop looking at it.

## Observability

- `journalctl -u X -f` before anything fancier. It is already working.
- Loki's default `path_prefix` is `/tmp`, which is **cleared on reboot**.
- Your log shipper probably runs as a different user than the one that owns `0600` log
  files.
- Choose low-cardinality labels. A user ID as a label will destroy the index.
- `X-Request-ID {http.request.uuid}` costs one line and finds an exact request later.
- **Put your dashboard on a VPN, not a subdomain.** An admin panel that is not on the
  internet cannot be attacked from the internet.
- Your uptime monitor must run **somewhere other than your server**.
- Services that bind `0.0.0.0` and are protected only by the firewall are one
  `iptables -F` away from public.

## Proxy config

- **An unknown Caddy placeholder is not an error, it is emitted as literal text.** Mine
  shipped `x-request-id: {http.request.id}` for weeks. Verify with
  `curl -sI https://yoursite/`.

## The general ones

- Three layers must agree for a packet to arrive: bind address, host firewall, cloud
  firewall. They fail independently and silently.
- If it works interactively and fails from a machine, it is PATH or environment. Every
  time.
- Templates drift from live config. Wildcards hide the drift.
- **Check confident advice against your own box.** It is usually one command. I was told
  `nohup` forks a child and that this explained a PID mismatch. It does not fork; it
  `exec`s. The actual bug was a stale process from an hour earlier holding the port.
- The answer to "should I bind `0.0.0.0`?" changes when you add a proxy. Advice has a
  timestamp and an assumed stack; check which one it was written for.
- Write down the things you found wrong with your own setup. This series has a list, and
  writing it was more useful than any of the configuration.
