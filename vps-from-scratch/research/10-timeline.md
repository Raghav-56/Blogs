# Research: the timeline

`git log` on the site repo, read as a narrative. Every stage in the series maps to real
commits, which is what makes the series honest rather than reconstructed.

```
2026-06-28  530b708  feat: initial commit
2026-06-29  5f84340  feat: Add favicon
2026-06-29  23684ec  use oxmgr
2026-06-29  06581bf  feat: upadate for bun native use
2026-06-29  5a36b7f  fix: update for sep deploy and dev
2026-06-29  4fb2857  feat(docs): add readme with todo
2026-06-29  d8b5d1c  feat: add deploy workflow (#1)
2026-06-29  b339423  fix: deployment workflow (#2)
2026-06-30  464de4c  feat: implement health check
2026-06-30  f23545b  feat: improve deployment workflow
2026-06-30  0eec14c  chore: improve aesthetics of deploy workflow discord
2026-06-30  15d1e78  feat: add logging for loki
2026-06-30  a20fe23  fix: deploy workflow, dev behind auth
2026-07-11  e5ffe4e  dev: add skills
2026-07-12  3aea7e0  feat(build): introduce template-driven static site build pipeline
2026-07-12  c73fe17  feat(infra): rewrite server routing and add dev/deploy lifecycle scripts
2026-07-12  fb5d46b  docs: restructure documentation, agent guidelines, and skill references
2026-07-12  9a8fcbd  fix(ci): update health check endpoints to target production caddy and bun API
2026-07-12  fbaaaa6  feat: add direct resume pdf download for terminal clients
2026-07-12  a038e02  docs: add git commits guidelines to AGENTS.md
2026-07-12  15229ad  feat: serve text resume to terminal clients using glow ...
2026-07-13  964ab00  fix: repair health-check propagation and unroutable /health endpoint
2026-07-13  5ae0035  feat: harden production Caddy config and add site discoverability
2026-07-13  ad05f40  feat: add multi-theme browser UI with Flexoki palette
2026-07-13  62d4a80  feat: self-host Fira Mono and Inter fonts
2026-07-13  4422865  feat: build resume assets from git submodule instead of remote fetch
2026-07-13  5926dda  feat: migrate browser site build to Astro with Tailwind v4
2026-07-13  6c2b0b1  feat: add about and projects pages via content collections
2026-07-13  726d08e  feat: add blog with RSS feed and generated sitemap
2026-07-14  88ffaa1  feat: add contact form with Bun email endpoint and full terminal parity
2026-07-14  61c2e47  docs: update architecture docs for Astro migration and new pages
2026-07-14  ...      (docs, submodule, content commits)
2026-07-15  b1b5125  feat: content pipeline, 16 project pages, and timeline
2026-07-15  6228923  feat: admin mode - content editing service on admin subdomain
2026-07-15  c9ef639  fix: rename admin server to .tsx so Bun parses its JSX
2026-07-15  0e7437e  fix: pin hono/jsx via per-file pragma
2026-07-15  875df14  ci: fail deploy when server git pull fails; require health check
```

## The shape of it

**Day 1 (28 Jun): it exists.** Initial commit. On the server, the artifact from this
phase is `~/raghav/dprc/raghav56.tech.service`: `node server.js` under systemd, with
`AmbientCapabilities=CAP_NET_BIND_SERVICE` commented out. Also from this era: nginx
serving the site, and `/var/www/html/index.nginx-debian.html` still on disk today.

**Day 2 (29 Jun): supervision and automation, same day.** `use oxmgr` and
`feat: add deploy workflow` are 24 hours after the initial commit, with a `fix:
deployment workflow (#2)` immediately after. Nobody gets the pipeline right first time.

**Day 3 (30 Jun): observability.** `implement health check` then `add logging for loki`.
The health check comes before any of the site's actual features, which is the correct
instinct.

**11 day gap, then a rebuild (12 to 15 Jul).** The build pipeline is rewritten, the
browser site migrates to Astro, and the terminal rendering is generalized from one card
to every route. Note `fix: repair health-check propagation and unroutable /health
endpoint` on 13 Jul: the health check itself was broken, which is the joke that writes
itself.

**Last commit: `ci: fail deploy when server git pull fails; require health check`.** The
dirty-tree guard. The most recent thing learned was about the deploy pipeline, not the
site.

## What the timeline is good for in the writing

- It proves the progression is real. The series is not "here is the best practice", it is
  "here is the order I actually learned these in, and the failure that taught each one".
- The stage order in the chapters matches the commit order: bare process, systemd,
  proxy, TLS, domain, supervisor, CI, observability.
- It licenses honesty about the loose ends, because they are visible in the log. The
  admin service lands on 15 Jul and is still not running in production.

## The layers, as three exhibits on one machine

The box happens to contain all three stages simultaneously, and all three are quotable:

1. **Baseline.** Stock nginx welcome page still in `/var/www/html/`. The archived
   `node server.js` unit with the capability line commented out.
2. **Working but wrong.** A system unit running `npm run dev` in production, supervising
   a process four forks removed from the real server, bound to `0.0.0.0` with a firewall
   hole punched for it.
3. **Current.** Bun on loopback only, supervised with a health command, 23 days and zero
   restarts. Caddy as the sole public listener. Source tree mode 700 and unreadable by
   the web server. Config templates committed beside gitignored secrets. `caddy validate`
   before reload. A dirty-tree guard before the pull. SHA-pinned actions. A health check
   that spoofs a curl User-Agent to prove the actual feature works.
