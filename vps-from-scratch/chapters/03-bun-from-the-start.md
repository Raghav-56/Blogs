# 03. bun from the start

You install a runtime, everything works, you write a service file, and the log says:

```
bun: command not found
```

You SSH in and type `bun --version` and it prints `1.3.14`. It is right there. The service
still cannot find it.

This chapter is about the runtime, and mostly about that.

## Why bun

This box runs Bun 1.3.14. Node 22 is also installed from apt, mostly as a dependency of
other things. The site's `package.json` runs everything through bun:

```json
{
  "scripts": {
    "dev": "bun --bun astro dev",
    "build": "bun --bun astro build",
    "preview": "bun --bun astro preview"
  }
}
```

Reasons that hold up in practice:

- **It is one binary.** 91 MB, no `node_modules` for the runtime itself, no version
  manager. Compare with the nvm situation on this same box: nvm is installed, sourced by
  `.bashrc` on every single shell start, and has **zero Node versions installed** because
  `node` comes from apt instead. It is pure startup cost achieving nothing. That kind of
  drift does not happen with a single binary.
- **`bun install` is fast enough to change behaviour.** On a free-tier machine, install
  time is the difference between iterating and waiting.
- **It runs TypeScript and JSX directly.** No build step, no `ts-node`, no compile
  pipeline for a small server.
- **`bun run --hot` restarts on file change** without a separate watcher process. Chapter
  05 shows what happens when you use `nodemon` in production instead, and it is not good.
- **The standard library is batteries-included.** `Bun.serve`, `Bun.file`, an SQLite
  driver, a test runner, a `.env` loader, all built in.

Install:

```bash
curl -fsSL https://bun.sh/install | bash
```

It lands in `~/.bun/bin/bun` and appends a PATH export to your shell config. ARM64 is a
first-class target, so the aarch64 box gets a native build.

`--bun` in those scripts is worth knowing: tools like Astro shell out to `node` by
default. `bun --bun astro build` forces the whole thing onto the Bun runtime.

## Now the part that actually matters

The installer added this to `~/.bash_profile`:

```bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

Go back to the table in chapter 02. `~/.bash_profile` is read by **login shells only**.
So on this machine, bun is on PATH when you SSH in, and is not on PATH for:

- **systemd services.** They start with a minimal environment.
- **A process manager daemon.** Same.
- **`ssh host "command"`**, which is how CI deploys. Non-interactive, no startup files.
- **cron**, if you use it. Its PATH is famously `/usr/bin:/bin` and nothing else.

Same binary, same machine, same user. Invisible to all four.

## The three fixes, and when to use each

**1. Absolute paths in anything a machine runs.** This is the right answer for service
definitions. From this box's process manager config:

```toml
command = "/home/ubuntu/.bun/bin/bun run --hot server.js"
```

Not `bun run --hot server.js`. It looks clumsy and it is correct: it does not depend on
any environment being set up, so it works the first time and keeps working.

**2. Set PATH explicitly in the unit.** For a systemd service:

```ini
Environment="PATH=/home/ubuntu/.bun/bin:/usr/local/bin:/usr/bin:/bin"
```

**3. Export it at the top of remote scripts.** From this box's deploy workflow, the very
first thing that runs over SSH:

```bash
export PATH="$HOME/.bun/bin:$HOME/.cargo/bin:$HOME/.local/bin:/usr/local/bin:/usr/bin:$PATH"
```

That line exists because of exactly this problem. A deploy that works perfectly when you
paste the commands into an SSH session fails in CI with `bun: command not found`, and the
difference is invisible until you know about it.

Do not solve this by symlinking bun into `/usr/local/bin`. It works, and then you upgrade
bun and the symlink points at nothing, and you have hidden the dependency instead of
declaring it.

## The other environment trap

Bun loads `.env` automatically when you run it yourself. Under a supervisor, do not rely
on it. The comment in this box's process config says it plainly:

```toml
# Env vars must be explicit — OxMgr daemon runs in a clean env,
# Bun does NOT auto-load .env when launched by a process manager.
# HOSTNAME is set explicitly to override the system hostname env var.
```

Three separate problems compressed into three lines:

1. A supervisor daemon runs in a clean environment. Nothing from your shell reaches it.
2. `.env` auto-loading depends on the working directory and on the runtime's behaviour.
   Declare your variables in the service config where you can see them.
3. **`HOSTNAME` is already set by the system** to the machine's hostname. On this box that
   is `oracler`. If your app does `process.env.HOSTNAME || "127.0.0.1"` to pick a bind
   address, it will try to bind to `oracler` and you will get an error that makes no sense
   at all. That override exists because it happened.

That third one generalises: before you name an environment variable, check that the
system does not already own it. `HOSTNAME`, `USER`, `HOME`, `PATH`, `SHELL`, `TERM`,
`LANG` are all taken. Prefix your own: `APP_PORT`, not `PORT`, if you want to be careful.

## A server you can actually deploy

```js
const PORT = parseInt(process.env.PORT || "3000", 10);
const NODE_ENV = process.env.NODE_ENV || "development";
const HOSTNAME = process.env.HOSTNAME || (NODE_ENV === "production" ? "0.0.0.0" : "127.0.0.1");

Bun.serve({
  port: PORT,
  hostname: HOSTNAME,
  fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/health") return new Response("ok");
    return new Response("hello");
  },
});
```

That is close to the real `server.js` on this box, and it has one flaw worth pointing at
now because chapter 06 is entirely about it: the production default is `0.0.0.0`, which
means every network interface, which means the public internet. It is only loopback in
practice because the supervisor sets `HOSTNAME=127.0.0.1` explicitly.

A safe value in your config beats a clever default in your code. But an unsafe default in
your code is a trap waiting for the day someone runs the app without the config.

**Always ship a `/health` endpoint.** It costs two lines. Chapters 12 and 13 both depend
on it: one to decide whether to restart the process, one to decide whether the deploy
worked. Add it before you need it.

## Development on the server

One workflow note that saves real time. This box's own contributor docs say:

> **Development happens on the server, not locally.** If you find yourself on Windows
> wanting to build or test: push the code to GitHub to sync it, then `ssh oracler`
> followed by `z raghav56.tech`.

Building where you deploy eliminates an entire category of "works on my machine".
Combined with `tmux` and a shell you have actually configured, the server becomes the
development machine rather than a place you throw artifacts at.

Next: putting it on the internet, badly.
