# 06. binding and firewalls

Your app is running. systemd says `active (running)`. The logs say
`Listening on port 8080`. You open `http://<YOUR_SERVER_IP>:8080` and the browser spins
forever, then times out.

Nothing is broken. Three separate systems have to agree before a packet reaches your
process, and they fail silently and independently. If you learn one thing from this whole
series, make it this chapter.

## The three layers

1. **What address your app bound to.** A kernel-level decision your code made.
2. **The host firewall.** iptables on this machine.
3. **The cloud network.** Oracle's VCN security list, which is completely invisible from
   inside the box.

Any one says no and you get a hang. None of them logs anything by default.

## Layer 1: what "listening on port 8080" actually means

A socket is bound to an **address and port**, not just a port. The address decides which
network interfaces can deliver packets to it.

```
$ ss -tlnp
State   Local Address:Port    Process
LISTEN     127.0.0.1:2019     caddy      (admin API)
LISTEN     127.0.0.1:2056     bun        (site API)
LISTEN     127.0.0.1:8978     docker-proxy
LISTEN             *:80       caddy
LISTEN             *:443      caddy
LISTEN             *:5001     node       (team API)
LISTEN             *:3000     grafana
LISTEN       0.0.0.0:22       sshd
```

Read the **Local Address** column, not the port. That column is the entire lesson.

- **`127.0.0.1:2056`** binds the loopback interface only. The kernel will deliver packets
  that originated on this machine and physically nothing else. A packet from the internet
  cannot reach it. Not "is blocked from reaching it": cannot. No firewall involvement at
  all.
- **`0.0.0.0:22`** and **`*:80`** mean every interface, including the public one. `*` is
  what `ss` prints when a socket is bound to both IPv4 and IPv6 wildcards.

So on this box:

| Service | Bind | Reachable from the internet |
|---|---|---|
| Site API (Bun) | `127.0.0.1:2056` | no, only through the proxy |
| Team API (Node) | `*:5001` | **yes, directly** |

Same machine. Same kind of app. Completely different exposure, decided by one argument.

## How each one got that way

The Node API:

```js
const PORT = process.env.PORT || 5001;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

`app.listen(port)` with no host argument defaults to all interfaces. That is Express's
documented behaviour, and it is the single most common accidental exposure in student
projects, because every tutorial writes it that way.

The safe version is one argument:

```js
app.listen(PORT, "127.0.0.1");
```

Because port 5001 is also open in this box's firewall (see below), that API answers
plaintext HTTP on `http://<YOUR_SERVER_IP>:5001`. It has a perfectly good HTTPS front
door through the proxy, and it also has a back door with no TLS, no access logging, and
none of the proxy's headers. Anyone who finds the port talks straight to the app.

The Bun server on the same box has a more careful version, and it is still not quite
right:

```js
const HOSTNAME = process.env.HOSTNAME || (NODE_ENV === "production" ? "0.0.0.0" : "127.0.0.1");
```

Loopback in development, all interfaces in production. In practice it binds loopback,
because the process manager sets `HOSTNAME=127.0.0.1` explicitly in its config. So the
running system is correct, and the code's default is not. The day someone runs that app
without the supervisor, it is public.

**A safe value in your config beats a clever default in your code. An unsafe default in
your code is a trap with a timer on it.**

### When `0.0.0.0` is the right answer

Important caveat, because this chapter will otherwise strand you.

If you have **no reverse proxy yet**, `0.0.0.0` is correct and `127.0.0.1` will drive you
mad. Loopback means nothing outside the machine can reach it, including you, from your
laptop. On my first server that was exactly the confusion:

```
INFO:     Uvicorn running on http://localhost:8086
```

Port opened in the cloud console, port opened in the firewall, and still nothing, because
the app was bound to loopback and no proxy existed to bridge the gap. The fix at that
stage was `host="0.0.0.0"`, and it was the right fix.

Chapters 07 and 10 change the answer. Once a proxy owns 80 and 443 and terminates TLS,
every backend goes back to loopback and the proxy is the only thing exposed.

So:

| Stage | Bind | Why |
|---|---|---|
| No proxy, just getting online | `0.0.0.0` + firewall rule | nothing else can reach it |
| Proxy in front (from chapter 07 on) | `127.0.0.1` | the proxy is the only public door |

Same question, opposite correct answers, depending on what else is running. If a tutorial
tells you to bind `0.0.0.0`, check which of these two worlds it is written for.

### Why you have two IP addresses

One more thing that confuses everyone on Oracle specifically:

```
$ ip addr show
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 ...
    inet 10.0.0.228/24 metric 100 brd 10.0.0.255 scope global ens3

$ curl ifconfig.me
140.245.219.236
```

Two different addresses, and the machine only knows about one of them.

`10.0.0.228` is the **private** address inside Oracle's virtual network. It is the only
one the operating system can see, and the only one you can bind to. `140.245.219.236` is
the **public** address, which lives on Oracle's gateway and is mapped to your private
address by NAT.

So `ip addr show` will never show your public IP, and you cannot bind to it. This is why
`0.0.0.0` matters: it means "every interface I have", which includes the private address
that public traffic actually arrives on after translation. Binding to the public IP
literally fails, because the machine does not have it.

Find your public IP in the console, or with `curl ifconfig.me`.

## Layer 2: the firewall, and the Oracle surprise

Every tutorial you will read says:

```bash
sudo ufw allow 80
```

On this Oracle image, `ufw` **is not installed**. That command does not exist. Rules are
raw iptables, persisted by `netfilter-persistent`.

```
$ sudo iptables -L INPUT -n
Chain INPUT (policy DROP)
target  prot  opt  source     destination
ts-input  all  --  0.0.0.0/0  0.0.0.0/0
ACCEPT    all  --  0.0.0.0/0  0.0.0.0/0     (in: lo)
ACCEPT    all  --  0.0.0.0/0  0.0.0.0/0     ctstate RELATED,ESTABLISHED
ACCEPT    tcp  --  0.0.0.0/0  0.0.0.0/0     tcp dpt:22
ACCEPT    tcp  --  0.0.0.0/0  0.0.0.0/0     tcp dpt:80
ACCEPT    tcp  --  0.0.0.0/0  0.0.0.0/0     tcp dpt:443
ACCEPT    tcp  --  0.0.0.0/0  0.0.0.0/0     tcp dpt:5001
```

**Policy DROP.** Anything not explicitly accepted is discarded with no response at all.

That is a good default and it explains the symptom at the top of this chapter. A `DROP`
sends nothing back, so your client waits for a timeout. A `REJECT` would answer
immediately with "connection refused".

**This is your best diagnostic.** Learn to tell them apart:

| Symptom | Meaning |
|---|---|
| Hangs, then times out | A firewall is dropping it. Layer 2 or layer 3. |
| Instant "connection refused" | Reached the machine; nothing is listening on that port. Layer 1. |
| Connects, no response | It is listening, and your app is broken. Not a network problem. |

To open a port and keep it after reboot:

```bash
sudo iptables -I INPUT 4 -p tcp --dport 8080 -j ACCEPT
sudo netfilter-persistent save
```

`-I INPUT 4` inserts at position 4. **Position matters.** If you `-A` (append) after a
DROP rule, your rule sits below it and will never match. Check with
`sudo iptables -L INPUT -n --line-numbers`.

## The Oracle chain you must never delete

```
Chain OUTPUT
-A OUTPUT -d 169.254.0.0/16 -j InstanceServices

Chain InstanceServices
-A InstanceServices -d 169.254.0.2/32 -p tcp --dport 3260 -m owner --uid-owner 0 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p tcp --dport 80 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp --dport 53 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp --dport 67 -j ACCEPT
-A InstanceServices -d 169.254.169.254/32 -p udp --dport 123 -j ACCEPT
-A InstanceServices -d 169.254.0.0/16 -p tcp -j REJECT --reject-with tcp-reset
```

`169.254.169.254` is the cloud metadata endpoint. Port 3260 is **iSCSI, which is how your
boot volume is attached**.

The internet's most common firewall advice is "just flush the rules and start over":

```bash
sudo iptables -F      # DO NOT DO THIS ON ORACLE CLOUD
```

That breaks metadata, DHCP, NTP, and on some shapes the root disk. It is the most
expensive single command available to you on this platform. If you want a clean slate,
edit `/etc/iptables/rules.v4` by hand and keep the `InstanceServices` chain intact.

## Layer 3: the one you cannot see from the server

Oracle's VCN security list (or a Network Security Group) sits in front of the host
firewall. **Nothing on the machine tells you what it says.**

The default VCN opens port 22 and nothing else. So the classic sequence is: student
starts a web server, opens port 80 in iptables, confirms `ss` shows it listening, confirms
`curl localhost` works, and still gets nothing from outside. Everything on the box is
correct. The packet never arrives.

Fix it in the console: Networking, then your VCN, then the subnet's Security List, then
add an ingress rule. Source `0.0.0.0/0`, TCP, destination port 80 and 443.

## The debugging order that always works

Work outwards. Each step eliminates one layer.

```bash
# 1. Is the app alive at all?
curl -I http://127.0.0.1:8080
#    fails → your app is the problem, stop here

# 2. Is it bound beyond loopback?
ss -tlnp | grep 8080
#    shows 127.0.0.1 → your app binds loopback. If that is intentional, put a
#    proxy in front. If not, fix the listen() call.

# 3. Does the host firewall allow it?
sudo iptables -L INPUT -n --line-numbers | grep 8080

# 4. Nothing wrong on the box and still hanging?
#    It is the cloud security list. Go to the console.
```

Four commands. They will resolve this class of problem essentially every time.

## Tailscale, and the pattern worth copying

Notice the first rule in that INPUT chain, before every port ACCEPT:

```
-A ts-input -i tailscale0 -j ACCEPT
-A ts-input -s 100.64.0.0/10 ! -i tailscale0 -j DROP
```

Anything arriving over the Tailscale tunnel is accepted regardless of port. Tailscale is
a mesh VPN that takes about two minutes to set up and gives every device you own a
private, stable address.

This box uses it as the **admin plane**. Grafana on port 3000 and Loki on 3100 are never
exposed publicly, have no subdomain, and need no password gateway. They are simply
unreachable unless you are on the tailnet. That is a much better answer than "put a login
page in front of it", because there is no login page to have a bug in.

Adopt this early. Your database admin panel, your metrics dashboard, your staging
environment: none of those need to be on the public internet. Put them on the tailnet.

Honest caveat, from this same box: Grafana and Loki still **bind** `*`. They are protected
by the firewall, not by their own configuration. If someone flushes iptables while
debugging, they are instantly public. Binding them to the tailnet address would be belt
and braces. Defence by omission is defence until someone changes the omission.

## Docker rewrites your firewall

This one catches experienced people.

**`docker run -p 8080:80` publishes to all interfaces and bypasses your INPUT rules.**
Docker writes its own iptables rules, and the port forward happens in `PREROUTING`, before
the INPUT chain is ever consulted. Your carefully configured DROP policy does not apply.

People expose databases to the entire internet this way and never find out.

The fix is a prefix, and this box's pgAdmin container has it:

```yaml
ports:
  - "127.0.0.1:8978:80"     # NOT "8978:80"
```

That produces a DNAT rule scoped to `127.0.0.1`, so the container is reachable only from
the machine, and then only through the proxy that fronts it with authentication. The
Postgres container next to it publishes **no ports at all** and is reachable only on the
compose network. That is the right default for a database.

Check yourself: after starting any container, run `ss -tlnp` and see what appeared.

## A habit worth forming

Once a month, run:

```bash
ss -tlnp
```

and for each line ask: what is that, and does it need to be listening where it is
listening? On this box that exercise turns up `rpcbind` on `0.0.0.0:111`, an NFS
dependency that came with the image and is used by nothing. Firewalled, so harmless, but
it is free attack surface:

```bash
sudo systemctl disable --now rpcbind rpcbind.socket
```

Fewer things listening is the cheapest security improvement available.

Next: you have two apps and one port 443.
