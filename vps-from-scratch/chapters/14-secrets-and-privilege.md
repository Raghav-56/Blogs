# 14. secrets and privilege

You committed the API key. Everybody does it once.

The bad news: `git rm` does not help. It is in the history, and history is what gets
cloned. GitHub's secret scanning may email you within minutes; bots watching the public
event firehose are often faster. Assume any key that touched a public repo is compromised,
and rotate it rather than trying to hide it.

Setting up so it cannot happen is a fifteen-minute job.

## Gitignore the real thing, commit the shape

```
# .gitignore

# secrets
.env

# server configs - provided *.template files for reference
oxfile.toml
raghav56.tech.caddy
raghav56.tech.caddyfile

# build output
dist/
node_modules/
.astro/
```

Then commit a template beside each ignored file. From this box:

```
$ git ls-files | grep -E 'oxfile|caddy|env'
.env.example
oxfile.toml.template
raghav56.tech.caddy.template
```

The template documents structure without leaking values:

```toml
[apps.env]
NODE_ENV = "production"
PORT = "{{BUN_PORT}}"
HOSTNAME = "127.0.0.1"
# Contact form email delivery (https://resend.com API key).
# Without it, /api/contact logs the message and returns 503.
RESEND_API_KEY = "{{RESEND_API_KEY}}"
```

This solves the thing gitignoring alone does not: **a new person, or you in six months on
a new machine, can see exactly which variables exist and what they are for.** Without a
template, a gitignored config file is invisible knowledge, and setting up the project
becomes an archaeology exercise.

Note the comment explaining what happens when the key is *absent*: the endpoint logs the
message and returns 503. Documenting graceful degradation means someone can run the
project without every credential.

The `.env.example` on this box goes further and explains the values:

```bash
# Bind address. Keep loopback-only — Caddy is the sole public-facing process.
HOSTNAME=127.0.0.1

# Local Bun port. dev.sh defaults to 3056; production uses 2056 via OxMgr.
PORT=3056
```

## Where secrets actually live

Three places, and only three:

**1. The process manager or service config on the server**, gitignored, for runtime
secrets:

```toml
[apps.env]
RESEND_API_KEY = "<the real key>"
```

**2. Your CI provider's secret store**, for deploy-time credentials. On this project:
`GH_PAT`, `HOST`, `USERNAME`, `PORT`, `SSH_KEY`, `DISCORD_WEBHOOK`.

**3. A `.env` file on your dev machine**, gitignored.

Never in the code. Never in a Docker image. Never in a CI log (be careful with `set -x` in
a script that handles secrets, because it prints every expansion).

## Gitignored is not the same as safe

Here is the uncomfortable part, and it is live on this box.

The real `oxfile.toml` is correctly gitignored and has never been committed. It also
contains a live API key in cleartext, mode `664`, on a machine with several user accounts
on it.

```
-rw-rw-r--  1 ubuntu ubuntu  1263  oxfile.toml
```

That last `r` is "everyone else on this machine". Every other user can read the key.
Gitignoring protects you from GitHub. It does nothing about the filesystem.

```bash
chmod 600 oxfile.toml .env
```

Owner read and write, nobody else. Do this on every file containing a credential, and
check it periodically:

```bash
find ~ -name '.env' -o -name '*.toml' | xargs ls -l
```

The same applies to server logs. This box's proxy log files are `0600 caddy:caddy`, which
is correct, and which is also why the log shipper (running as a different user) needs
thought. Restrictive permissions create real friction. That is what they are for.

## Privilege separation, and the 403 that caused it

The other half of "who can read what" is "what can the web server reach".

I did not design this part. I hit it.

First attempt at serving the site, the proxy config pointed straight at the repo:

```caddyfile
root * /home/ubuntu/raghav/raghav56.tech/
```

and every request returned:

```
$ curl -v https://raghav56.tech
< HTTP/2 403
< server: Caddy
< content-length: 0
```

A 403 with **zero bytes of explanation**. TLS was fine. Routing was fine. The file
existed. `ls -l` said it was world readable:

```
-rw-rw-r-- 1 ubuntu ubuntu index.txt
```

### The command that solves this in one line

```
$ namei -l /home/ubuntu/raghav/raghav56.tech/static/index.txt
f: /home/ubuntu/raghav/raghav56.tech/static/index.txt
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu          ← 750
drwx------ ubuntu ubuntu raghav          ← 700, and there it is
drwxrwxr-x ubuntu ubuntu raghav56.tech
drwxrwxr-x ubuntu ubuntu static
-rw-rw-r-- ubuntu ubuntu index.txt
```

`namei -l` walks every component of a path and prints its permissions. Learn it now; it
turns a whole category of "permission denied on a file that is obviously readable" into a
five-second answer.

The rule it makes visible: **directory permissions compose.** To read a file you need
execute (traverse) on *every* directory above it. `index.txt` being world readable is
irrelevant when `/home/ubuntu/raghav` two levels up is `700`. `ls -l` on the file tells
you nothing useful, which is exactly why the error is so confusing.

The obvious fix is `chmod 755` on your home directory. Do not do that. It makes every
project, key, and dotfile in your home directory readable by every account on the machine,
to fix one file.

The right fix is the boundary:

| Path | Owner | Mode | Web server access |
|---|---|---|---|
| `/home/ubuntu/raghav/raghav56.tech/` | `ubuntu:ubuntu` | `700` | **none** |
| `/var/www/raghav56.tech/` | `ubuntu:www-data` | `755` | read-only |

The proxy runs as user `caddy`. The source tree is mode `700` owned by `ubuntu`, so the
proxy **cannot read it at all**. Not "is configured not to". Cannot.

The build compiles into `dist/` and rsyncs the compiled output across the boundary into
`/var/www/`. The public web root contains HTML, CSS, images, and nothing else.

The 403 was not a problem to work around. It was the filesystem telling me the
architecture was wrong.

Why this matters: a path traversal bug in any file server, yours or someone else's, is the
difference between leaking your compiled HTML (which is already public) and leaking your
`.env`, your `.git` directory, and your SSH keys. The `.git` one is the real killer:
directories are frequently exposed by accident, and a `.git` directory contains your
entire source history, including that key you removed three commits ago.

**Never point a web server at a directory that contains your source code.** Build to a
separate directory and serve that.

Concretely:

```bash
sudo mkdir -p /var/www/mysite
sudo chown ubuntu:www-data /var/www/mysite
chmod 755 /var/www/mysite
chmod 700 ~/myproject
```

## A pre-commit hook, for the day you are tired

A teammate's project on this box installs git hooks that block a commit when it detects:

- a `.env` file being committed
- `console.log` left in the source
- patterns that look like hardcoded secrets

A minimal version, `.git/hooks/pre-commit`:

```bash
#!/usr/bin/env bash
if git diff --cached --name-only | grep -qE '(^|/)\.env$'; then
    echo "refusing to commit a .env file" >&2
    exit 1
fi
```

```bash
chmod +x .git/hooks/pre-commit
```

Hooks are not committed with the repo, so provide a `setup-hooks.sh` that copies them from
a tracked directory into `.git/hooks/`. Or use a tool like `pre-commit` that manages this
properly.

Also enable GitHub's push protection in repository settings. It blocks pushes containing
recognisable credential formats, and it has saved more people than any hook.

## When it happens anyway

1. **Rotate the key first.** Not after cleaning history. Immediately. The key is
   compromised the moment it is pushed and every second of delay is exposure.
2. Then remove it from history if you like, with `git filter-repo` or BFG. Understand that
   this rewrites every commit hash and requires a force push, and that anyone who already
   cloned still has the old history. This step is hygiene, not remediation.
3. Check what the key could reach. An email-sending key means someone can send mail as
   you. A cloud key can mean a bill.

Rotation is the fix. History cleaning is tidying up.

## A short audit worth running today

```bash
# what secrets exist and who can read them
find ~ -maxdepth 3 \( -name '.env*' -o -name '*.toml' -o -name '*credentials*' \) -exec ls -l {} \;

# is anything sensitive tracked in git
git ls-files | grep -iE 'env|secret|credential|key'

# what is your web server actually able to serve
ls -la /var/www/

# what is listening, and does it need to be
ss -tlnp
```

Four commands. Run them on your own box now rather than after something goes wrong.

Next: knowing your site is down before someone tells you.
