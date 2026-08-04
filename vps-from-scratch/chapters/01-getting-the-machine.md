# 01. getting the machine

You want to put something on the internet. Every tutorial starts with "get a VPS from
DigitalOcean for $6 a month", and you are a student, and $6 a month is a real number.

There is a better option, and almost nobody uses it properly.

## Oracle Cloud Always Free

Oracle's free tier is not a trial. It does not expire after 12 months. The interesting
part is the ARM allocation:

**4 OCPUs and 24 GB of RAM**, which you can split across up to four instances or take as
one machine.

That is not a toy. For comparison, the $6/month box everyone recommends is 1 vCPU and
1 GB. This is four cores and twenty-four gigabytes, permanently, for nothing. Here is what
that actually looks like, read off the box this series is written on:

```
$ uname -a
Linux oracler 6.17.0-1009-oracle ... aarch64 aarch64 aarch64 GNU/Linux

$ nproc
4

$ free -h
               total        used        free      shared  buff/cache   available
Mem:            23Gi       3.1Gi       686Mi        41Mi        20Gi        20Gi
Swap:             0B          0B          0B

$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       186G   32G  155G  18% /

$ uptime -p
up 12 weeks, 6 days, 5 hours
```

Four cores, 23 GiB, 186 GB of disk, and it has been up for three months. It runs a
website, a Postgres database, a pgAdmin container, a log aggregator, a metrics dashboard,
and a couple of other people's projects, and it is using 3 GB of RAM.

Read that `free` output carefully, because it is the first thing that confuses people.
`free` shows 686 Mi and `available` shows 20 Gi. The gap is page cache: the kernel filled
your idle memory with cached disk blocks and will hand it back the instant anything asks.
**Read the `available` column.** "Linux ate my RAM" is not a real problem.

Note also: **swap is 0**. Nothing is configured by default. On 23 GiB you will rarely
care, but it means an out-of-memory situation kills your process immediately rather than
degrading first. If you take a smaller instance, add a swap file.

## The catch, and it is a real one

Two of them.

**First: the signup wants a card.** It runs a small verification charge and refunds it.
You need a card that supports international transactions. This blocks some people
entirely, and it is worth knowing before you spend an hour on the form.

**Second, and this is the one everybody hits: "Out of capacity for shape
VM.Standard.A1.Flex".**

The free ARM capacity is genuinely, frequently exhausted in popular regions. You will
click Create and get that error. This is not a mistake you made.

What actually works:

1. **Choose your home region carefully at signup.** You cannot change it later, and it is
   the region where your free resources live. Pick a less-obvious one that is still
   geographically reasonable for you. If you sign up in a region with three data centres
   serving half a continent, you are queueing behind everyone else.
2. **Retry.** Capacity frees up constantly. Trying at different times of day, over a few
   days, usually gets you in. People write scripts that retry the API call on a loop; you
   do not need to go that far, but understand that the error is transient, not permanent.
3. **Ask for less.** A 1 OCPU / 6 GB instance often succeeds when 4/24 fails. You can
   edit the shape afterwards and scale it up, which frequently works when creating at
   full size does not.
4. **Take the AMD fallback if you must.** The always-free AMD shape (VM.Standard.E2.1.Micro)
   is 1/8 of an OCPU and 1 GB. It is weak, but it is a real Linux box with a real public
   IP, and everything in this series works on it. Start there and migrate to ARM when you
   get capacity.

At instance creation, one thing matters more than the rest: **paste your SSH public key
into the form**. Generate it first if you do not have one:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

There is no password login. If you skip this or paste the wrong key, you cannot get in,
and the fix involves the serial console. Take the extra thirty seconds.

Choose **Ubuntu 24.04 LTS**. LTS means five years of security updates and it is what every
piece of documentation on the internet assumes.

## Your machine is ARM, and that is a real fact about it

```
$ uname -m
aarch64
```

Not x86_64. Almost everything supports ARM now, but not everything, and when something
does not the error is rarely helpful.

What this means in practice:

- Downloading a prebuilt binary? Get the `arm64` or `aarch64` build, not `amd64`. Running
  an x86 binary on ARM gives you `cannot execute binary file: Exec format error`.
- Docker images need an `arm64` variant. Most official images have one. Some smaller
  projects publish x86 only, and you will find out when the container exits immediately.
- `apt install` is fine. Ubuntu builds everything for ARM.
- Anything you compile yourself is fine, and Rust and Go cross-compile cleanly.

Every hand-installed tool on this box is an ARM64 build. It has never actually been a
problem. But when something does break in a way that makes no sense, `uname -m` is worth
checking early.

The upside is real: ARM is why this is free, and Ampere cores are quick.

## First contact

```bash
ssh ubuntu@<YOUR_SERVER_IP>
```

The default user is `ubuntu` on Ubuntu images, not `root`. It has passwordless `sudo`.

The first thing to run, before anything else:

```bash
sudo apt update && sudo apt upgrade -y
```

The second thing is to notice what you have been given. Oracle's image ships two agents
you did not install (`oracle-cloud-agent` and a monitoring agent), and a firewall that is
already configured in a way that will confuse you within the hour. Chapter 06 covers that
properly, because it is the single largest source of "why can't I reach my app" on this
platform.

## What you should have now

- An always-free instance, ideally ARM, running Ubuntu 24.04 LTS.
- SSH access as `ubuntu` with a key.
- A public IP address, visible in the console.
- Packages updated.

If you open `http://<YOUR_SERVER_IP>` in a browser right now, nothing happens. That is
correct: nothing is listening yet, and even if something were, the network would not let
you through. Both halves of that get fixed, in that order.

Next: making a bare Ubuntu box somewhere you actually want to work.
