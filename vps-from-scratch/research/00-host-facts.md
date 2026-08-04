# Research: the host

Everything here was read off the live box on 2026-08-04. Commands are read-only.

## Machine

```
$ uname -a
Linux oracler 6.17.0-1009-oracle #9~24.04.1-Ubuntu SMP Sat Mar  7 01:08:51 UTC 2026 aarch64 aarch64 aarch64 GNU/Linux
```

| Item | Value |
|---|---|
| Hostname | `oracler` |
| Provider | Oracle Cloud Infrastructure, Always Free tier |
| Shape | VM.Standard.A1.Flex (Ampere ARM) |
| Arch | **aarch64** |
| Kernel | `6.17.0-1009-oracle` (Oracle-flavoured Ubuntu kernel) |
| OS | Ubuntu 24.04.4 LTS (Noble Numbat) |
| vCPU | 4 |
| RAM | 23 GiB usable, **0 B swap** |
| Disk | `/dev/sda1` 186 G, 32 G used, 155 G free (18%) |
| Uptime | 12 weeks, 6 days at time of writing |

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            23Gi       3.1Gi       686Mi        41Mi        20Gi        20Gi
Swap:             0B          0B          0B

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       186G   32G  155G  18% /
```

The "free" is 686 Mi and the "available" is 20 Gi. That gap is page cache, which the
kernel gives back on demand. Reading the wrong column is the classic panic.

Notes worth carrying into the chapters:

- The advertised free ARM allocation is 4 OCPU and 24 GB of RAM, splittable across up to
  four instances. This box takes the whole thing as one instance, which is why `nproc` is
  4 and RAM is 23 GiB (the missing ~1 GiB is firmware and kernel reserve).
- **Zero swap.** Nothing is configured by default. Builds are fine on 23 GiB, but an OOM
  here kills the process outright rather than thrashing first.
- Two Oracle agents run unasked: `snap.oracle-cloud-agent.*` and
  `unified-monitoring-agent*`. They are part of the image, not something that was
  installed.

## Toolchain versions

```
bun    1.3.14        /home/ubuntu/.bun/bin/bun
node   v22.23.2      /usr/bin/node        (apt, not nvm)
npm    10.9.8        bundled with system node
caddy  v2.11.4       /usr/bin/caddy
git    2.43.0
gh     2.45.0
oxmgr  0.5.0         /home/ubuntu/.cargo/bin/oxmgr  (cargo install from git)
```

Everything installed by hand on this box is an ARM64 build. `oxmgr` is
`ELF 64-bit LSB pie executable, ARM aarch64`. This is the recurring aarch64 tax: check
that a release exists for `aarch64` / `arm64` before you plan around a tool.

## What is publicly reachable

Exactly three ports answer from the internet by policy (22, 80, 443), plus one that was
opened by hand (5001). See `05-network.md`.

Everything else on this machine is either loopback-bound or reachable only over the
Tailscale tailnet.
