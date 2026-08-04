# Research: binding, firewall, and the three layers

For a packet from the internet to reach your app, three independent things must all
agree. Any one of them says no and you get a hang or a refusal, with no useful error.

1. The app must be **bound** to an address the packet can arrive on.
2. The **host firewall** must accept the port.
3. The **cloud network** (Oracle's VCN security list / NSG) must allow it, and that layer
   is completely invisible from inside the machine.

## Layer 1: what is actually listening

```
$ ss -tlnp
State  Recv-Q Send-Q     Local Address:Port   Process
LISTEN 0      4096          127.0.0.1:2019     caddy          (admin API)
LISTEN 0      512           127.0.0.1:2056     bun            (site API)
LISTEN 0      4096          127.0.0.1:8978     docker-proxy   (pgAdmin)
LISTEN 0      4096          127.0.0.1:12345    alloy
LISTEN 0      4096                  *:80       caddy
LISTEN 0      4096                  *:443      caddy
LISTEN 0      511                   *:5001     node           (team API)
LISTEN 0      4096                  *:3000     grafana
LISTEN 0      4096                  *:3100     loki
LISTEN 0      4096                  *:9096     loki           (gRPC)
LISTEN 0      4096            0.0.0.0:22       sshd
LISTEN 0      4096            0.0.0.0:111      rpcbind
LISTEN 0      4096         127.0.0.53:53       systemd-resolved
```

Read the **Local Address** column, not the port. That column is the whole lesson:

- `127.0.0.1:2056` means the kernel will only deliver packets that arrived on the
  loopback interface. A packet from the internet physically cannot reach it, no matter
  what the firewall says. This is the site's Bun API, and it is why the API is only
  reachable through Caddy.
- `*:5001` (equivalently `0.0.0.0:5001`) means every interface, including the public one.
- `ss -tlnp` needs sudo to show the process name; without it the Process column is blank
  for anything you do not own.

The two extremes on this box, side by side:

| App | Bind | Set where | Reachable from internet |
|---|---|---|---|
| Site API (Bun) | `127.0.0.1:2056` | `HOSTNAME=127.0.0.1` in the supervisor env | no, only via Caddy |
| Team API (Node) | `*:5001` | `app.listen(PORT)` with no host argument | **yes, directly** |

The Node code is:

```js
const PORT = process.env.PORT || 5001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

`app.listen(port)` with no host argument defaults to all interfaces. The correct call is
`app.listen(PORT, "127.0.0.1")`. Because port 5001 is also open in the firewall, that API
answers plaintext HTTP on `http://<YOUR_SERVER_IP>:5001`, bypassing Caddy, its TLS, and
its access logs entirely. Someone can talk to the API without ever touching the
certificate.

The site's Bun server shows the defensive version of the same code:

```js
const HOSTNAME = process.env.HOSTNAME || (NODE_ENV === "production" ? "0.0.0.0" : "127.0.0.1");
```

The default would still be `0.0.0.0` in production. It is loopback in practice only
because the supervisor sets `HOSTNAME=127.0.0.1` explicitly. Worth calling out: a safe
default in your config beats a clever default in your code.

## Layer 2: the host firewall

**`ufw` is not installed on this box.** Every tutorial says `sudo ufw allow 80`. On this
Oracle image that command does not exist. Rules are raw iptables, persisted by
`netfilter-persistent` from `/etc/iptables/rules.v4`.

```
Chain INPUT (policy DROP)
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 5001 -j ACCEPT
```

Default policy is `DROP`, which is the right default and also the reason a fresh Oracle
instance appears to hang rather than refuse when you hit an unopened port. A `DROP` gives
no response at all, so your browser spins until timeout. A `REJECT` would answer
immediately. When your service "isn't responding", the difference between a hang and an
instant connection-refused tells you which layer is saying no.

To persist a change:

```bash
sudo iptables -I INPUT 4 -p tcp --dport 8080 -j ACCEPT
sudo netfilter-persistent save        # writes /etc/iptables/rules.v4
```

Position matters. Appending with `-A` puts your rule after any earlier DROP and it will
never match.

## The Oracle-specific chain you must not delete

```
Chain OUTPUT
-A OUTPUT -d 169.254.0.0/16 -j InstanceServices

Chain InstanceServices
-A InstanceServices -d 169.254.0.2/32 -p tcp -m owner --uid-owner 0 -m tcp --dport 3260 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p tcp -m tcp --dport 80 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 53 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 67 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 69 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp -m udp --dport 123 -j ACCEPT
-A InstanceServices -d 169.254.0.0/16 -p tcp -j REJECT --reject-with tcp-reset
-A InstanceServices -d 169.254.0.0/16 -p udp -j REJECT --reject-with icmp-port-unreachable
```

`169.254.169.254` is the cloud metadata endpoint. Port 3260 is iSCSI, which is how the
**boot volume itself** is attached. The common advice "just flush iptables and start
clean" (`iptables -F`) breaks metadata, DHCP, NTP, and on some shapes the root disk. This
is the single most expensive mistake available on an Oracle instance. If you want a clean
slate, edit `/etc/iptables/rules.v4` and keep the `InstanceServices` chain.

## Layer 3: the invisible one

Oracle's VCN security list (or NSG) sits in front of everything above and **nothing on
the host tells you its state**. The default VCN allows 22 only. A student who opens port
80 in iptables and still gets nothing is almost always missing an ingress rule in the
console. Symptom: connection hangs, `ss` shows the process listening, iptables shows the
ACCEPT. Check the console.

Diagnostic order that actually works:

```bash
curl -I http://127.0.0.1:8080         # layer 1: is the app up at all
curl -I http://<PRIVATE_IP>:8080      # layer 1: is it bound beyond loopback
sudo iptables -L INPUT -n --line-numbers   # layer 2
# still nothing? layer 3, go to the OCI console
```

## Tailscale as the admin plane

```
-A ts-input -i tailscale0 -j ACCEPT
-A ts-input -s 100.64.0.0/10 ! -i tailscale0 -j DROP
-A ts-input -p udp -m udp --dport 41641 -j ACCEPT
```

`ts-input` is the **first** rule in the INPUT chain, before any of the port ACCEPTs. So
anything arriving over the tailnet is accepted regardless of port. That is how Grafana on
`*:3000` and Loki on `*:3100` are administered without ever being exposed publicly: no
firewall hole, no subdomain, no basic auth. Just not reachable unless you are on the
tailnet.

Honest caveat for the chapter: those services still **bind** `*`, so they are protected
by the firewall rather than by their own configuration. Defence by omission. If someone
later runs `iptables -F` to debug something, they are instantly public. Binding them to
the tailnet address or loopback would be the belt to the firewall's braces.

## Docker changes the rules under you

```
Chain DOCKER (nat)
DNAT tcp -- 0.0.0.0/0 -> 127.0.0.1 tcp dpt:8978 to:172.18.0.3:80
```

Docker writes its own iptables rules. This matters enormously: **`docker run -p 8080:80`
publishes to all interfaces and bypasses your INPUT rules**, because the DNAT happens in
`PREROUTING` before `INPUT` is consulted. People get burned by this constantly.

The correct form, which this box uses for pgAdmin:

```yaml
ports:
  - "127.0.0.1:8978:80"     # not "8978:80"
```

The DNAT above is scoped to `127.0.0.1` precisely because of that prefix. The Postgres
container publishes no ports at all and is reachable only on the compose network.

## Dead weight found

`rpcbind` listens on `0.0.0.0:111` and nothing uses it. Firewalled, so not exploitable
here, but it is an NFS dependency that came with the image. `systemctl disable --now
rpcbind rpcbind.socket` is a free reduction in attack surface. Good habit to teach:
periodically read `ss -tlnp` and ask "what is that, and do I need it".
