# 15. observability

Someone messages you: "hey, was your site down yesterday?"

You have no idea. You cannot check. There is no record.

That is the problem. The solution is a ladder, and most people should stop climbing about
a third of the way up.

## Rung 1: journalctl, which you already have

```bash
journalctl -u caddy -f                      # follow
journalctl -u caddy --since "1 hour ago"
journalctl -u caddy -p err                  # errors only
journalctl -u caddy -b                      # since last boot
journalctl --disk-usage
```

Anything a systemd service writes to stdout or stderr is here, timestamped, tagged, and
rotated, with no configuration. If you have a service and you cannot answer a question,
this is the first place to look.

`journalctl -b -1` shows the *previous* boot, which is how you find out why the machine
rebooted.

## Rung 2: real access logs, per site

The proxy has the traffic. Configure it to keep the traffic.

```caddyfile
log {
    output file /var/log/caddy/raghav56.log {
        roll_size 5mb
        roll_keep 5
        roll_keep_for 7d
    }
}
```

On this box that lives in a reusable snippet so every site gets it with its own filename
and rotation policy:

```caddyfile
import common-headers-logging "raghav56.log"     "5mb"   "5" "7d"
import common-headers-logging "dev_raghav56.log" "1.5mb" "2" "3d"
```

Production keeps 5 MB times 5 files for a week. Staging gets a fifth of that. Tuning per
site is a two-minute job and it is what stops logs from filling a disk. That failure mode
is real, it takes down everything at once, and it is entirely preventable.

Also worth adding, and present in that same snippet:

```caddyfile
header {
    X-Request-ID {http.request.uuid}
}
```

Every request gets a unique ID in its response headers. When a user reports a problem and
can give you the ID, you can find that exact request in the logs. Cheap, and hugely
useful the day you need it.

Verify it actually works, because an unknown placeholder in Caddy is emitted as literal
text rather than raising an error. Mine said `{http.request.id}` for weeks, which is not a
real placeholder, and every response carried that exact string as its "unique" ID:

```bash
curl -sI https://yoursite/ | grep -i request-id
```

Braces in the output mean the placeholder name is wrong. Chapter 10 has the details.

Caddy writes JSON, so `jq` is your query language:

```bash
# status code distribution today
jq -r '.status' /var/log/caddy/raghav56.log | sort | uniq -c | sort -rn

# every 5xx with its URI
jq -r 'select(.status >= 500) | "\(.ts) \(.status) \(.request.uri)"' /var/log/caddy/raghav56.log

# top paths
jq -r '.request.uri' /var/log/caddy/raghav56.log | sort | uniq -c | sort -rn | head -20
```

**Most people should stop here.** Journal plus rotated access logs plus `jq` answers
almost every question you will actually have on a single-server setup.

## Rung 3: a log pipeline

Grep across files stops working when you have several services and want to ask "what
happened at 14:32 across all of them".

This box runs Alloy (a log collector) shipping into Loki (a log database) with Grafana in
front. The collector config is short:

```river
loki.source.file "caddy_logs" {
  targets = [
    { __path__ = "/var/log/caddy/raghav56.log", job = "caddy", site = "raghav56", env = "prod" },
    { __path__ = "/var/log/caddy/dev_raghav56.log", job = "caddy", site = "dev_raghav56" },
    { __path__ = "/var/log/bun/app.log", job = "bun" },
  ]
  forward_to = [loki.write.local.receiver]
}

loki.write "local" {
  endpoint { url = "http://localhost:3100/loki/api/v1/push" }
}
```

The interesting part is the **labels**: `job`, `site`, `env`. Labels are what make this
queryable rather than just centralised:

```
{job="caddy", env="prod"} |= "500"
```

Loki indexes labels, not log content, which is why it runs comfortably on a small machine.
Choose labels with low cardinality: `site` and `env` are good, a user ID is a disaster.

Be honest with yourself about whether you need this. On a single server it is mostly
enjoyable rather than necessary. The genuine payoffs are correlating across services and
having a dashboard you can look at without SSH.

Two real gotchas from this box's setup, both worth knowing before you copy it:

**Loki's default storage path is `/tmp`.**

```yaml
path_prefix: /tmp/loki
```

`/tmp` is cleared on reboot. Your log history evaporates on the exact day you most want
it. Change it to `/var/lib/loki` before you put anything real in there.

**The collector runs as a different user than the one that owns the logs.** The proxy
writes `0600 caddy:caddy`; the collector runs as `alloy`. Either add the collector user to
the right group, or loosen the log file mode. This is the same friction from chapter 14,
arriving from the other direction: good permissions are inconvenient by design.

## Do not expose your dashboard

Grafana on this box listens on port 3000 and is **not** in the firewall's allow list. It
has no subdomain, no public certificate, and no login page facing the internet. It is
reachable only over Tailscale.

That is the pattern worth adopting. Look at the firewall rule ordering from chapter 06:

```
-A ts-input -i tailscale0 -j ACCEPT       ← first, before every port rule
-A INPUT -p tcp --dport 22 -j ACCEPT
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT
```

Anything on the tailnet reaches anything. Anything else reaches only 22, 80, and 443.

An admin panel that is not on the internet cannot be attacked from the internet. There is
no login form to have a bug in, no default password to forget, no CVE to patch under
pressure. Your metrics dashboard, database admin tool, and staging environment all belong
here.

Honest caveat, stated once more because it is this box's real weakness: Grafana and Loki
still **bind** `0.0.0.0`. They are protected by the firewall, not by their own
configuration. If someone flushes iptables while debugging, they are instantly public.
Binding them to the tailnet address or to loopback would be belt and braces. Defence by
omission is defence right up until somebody changes the omission.

## The thing that actually tells you

Everything above is for investigating after the fact. None of it tells you the site is
down right now.

This box's answer is a Discord webhook on every deploy, carrying deploy status, commit
info, and health check results, posted whether the deploy succeeded or failed. The script
that builds it has one detail worth copying:

```bash
jq -n --arg title "$TITLE" --argjson color "$COLOR" '{...}'
```

JSON built with `jq`, not by interpolating variables into a string. Hand-built JSON breaks
the first time a commit message contains a quote character.

For "is it up right now", add an external check. UptimeRobot's free tier polls a URL every
five minutes and emails you when it stops answering. Better options exist; the important
property is that **it runs somewhere other than your server**, because a monitor on the
box cannot tell you the box is gone.

Point it at your `/health` endpoint, the same one your process manager polls. One endpoint,
three consumers: the supervisor decides whether to restart, the deploy decides whether it
worked, the external monitor decides whether to wake you up.

## The realistic ladder

For a personal server, in order of value per hour spent:

1. **`journalctl`.** Free, already working.
2. **Rotated access logs per site plus `jq`.** Ten minutes of config.
3. **An external uptime monitor.** Five minutes, free, and it is the only thing here that
   proactively tells you something is wrong.
4. **Deploy notifications.** You already have the health check from chapter 13; wiring it
   to a webhook is a small step.
5. **Loki and Grafana.** Genuinely useful with several services, and a good thing to have
   built once so you understand what the hosted versions are doing. Not the first thing to
   build.

Steps 1 to 3 take under an hour and cover the overwhelming majority of "was my site down"
questions.

Next: everything else that will bite you.
