# 02. making the box yours

Fresh Ubuntu is a hostile place to work. No colours, no fuzzy matching, a prompt that
tells you nothing, and `git push` asks for a password that no longer exists.

You are going to spend a lot of hours in this shell. Half a day making it habitable pays
for itself in a week.

## SSH: stop typing the IP

On **your laptop**, not the server, edit `~/.ssh/config`:

```
Host oracler
    HostName <YOUR_SERVER_IP>
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

Now `ssh oracler` works. So does `scp file oracler:~/`. So does `rsync -a ./dist
oracler:/var/www/`. `ServerAliveInterval 60` sends a keepalive every minute so your
session does not silently die when your laptop's wifi hiccups.

## The SSH trick worth the whole chapter

Many campus networks, offices, and hotel wifi block outbound port 22. You cannot
`git clone git@github.com:...`, cannot push, and the error looks like the network is
simply broken.

GitHub runs the same SSH service on port 443. To a firewall it is indistinguishable from
HTTPS. Four lines, on the server's `~/.ssh/config`:

```
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
```

That is the real config on this box. Test it:

```bash
ssh -T git@github.com
# Hi Raghav-56! You've successfully authenticated...
```

Nothing else changes. Every `git clone`, `push`, and `pull` transparently uses 443.

## git, so it stops asking for a password

GitHub killed password authentication for git operations in 2021. If you are being asked
for a password, you are using an old workflow.

Install and authenticate the GitHub CLI:

```bash
sudo apt install gh
gh auth login       # choose SSH, let it upload your public key
```

That single command uploads your SSH key to your GitHub account, stores a token, and
configures git to use it. Afterwards:

```
$ gh auth status
github.com
  ✓ Logged in to github.com account Raghav-56
  - Git operations protocol: ssh
  - Token scopes: 'admin:public_key', 'gist', 'read:org', 'repo'
```

Then set your identity. `~/.gitconfig` on this box:

```ini
[init]
	defaultBranch = main
[user]
	name = Raghav Gupta
	email = 142162663+Raghav-56@users.noreply.github.com
[credential "https://github.com"]
	helper = 
	helper = !/usr/bin/gh auth git-credential
[core]
	editor = nvim
```

Two things there deserve explanation.

**Use the noreply email.** Every commit you make embeds an email address, permanently,
visible to anyone who clones the repo. Committing with your personal address publishes it
to scrapers forever. GitHub gives you a `<id>+<username>@users.noreply.github.com` alias
in Settings, and commits made with it still count towards your contribution graph. Set it
once, now, before you have a hundred commits with your real address in them.

**The empty `helper =` line is not a typo.** Git accumulates credential helpers from
system, global, and local config. Setting the helper to an empty string *clears* the
inherited list, so the `gh` helper on the next line is the only one. `gh auth login`
writes this for you. Worth recognising so you do not "clean it up" and break your auth.

## The three-file trap

This one costs everybody a day, eventually.

There are three shell startup files, and which of them runs depends on how the shell
started:

| File | Read when |
|---|---|
| `~/.bashrc` | interactive non-login shell (new terminal, a tmux pane) |
| `~/.bash_profile` | login shell (an SSH session, `su -`) |
| `~/.profile` | login shell, **only if `~/.bash_profile` does not exist** |
| none of them | **non-interactive**: systemd, cron, `ssh host "command"` |

Look at what that third row does. This box has a `~/.bash_profile`:

```bash
source ~/.bashrc
export KUBECONFIG=/home/ubuntu/.kube/config

# bun
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

Because that file exists, **`~/.profile` is never read**. Everything in it, including a
whole PATH block Ubuntu ships by default, is dead code on this machine. It took a while to
notice, because `.bashrc` happens to export the same directory independently.

And the fourth row is the one that will actually bite you: **systemd, cron, and a
non-interactive `ssh host "command"` read none of these files.** Every tool you installed
into your home directory is invisible to them. This is why, later in this series, a
service config hardcodes `/home/ubuntu/.bun/bin/bun` and a CI script re-exports PATH by
hand. It is not paranoia; it is the only thing that works.

The rule: **put your exports in `.bashrc`, make `.bash_profile` just
`source ~/.bashrc`**, and use absolute paths in anything a machine runs.

## Tools that make the terminal not painful

```bash
# a prompt that shows git branch, exit codes, and language versions
curl -sS https://starship.rs/install.sh | sh

# smarter cd: `z projname` jumps anywhere you have been
sudo apt install zoxide fzf ripgrep fd-find bat
```

On this box the Rust tools are cargo-installed: `bat` (cat with syntax highlighting),
`eza` (ls with colours and git status), `fd` (find that is actually usable), `rg`
(ripgrep, grep but fast enough to search a whole disk), `btm` (bottom, a readable `top`),
`zoxide`.

At the bottom of `~/.bashrc`:

```bash
eval "$(starship init bash)"
eval "$(zoxide init bash)"
```

`zoxide` alone changes how you use a shell. It remembers directories you visit, so after
visiting your project once, `z raghav56` gets you there from anywhere.

## Three functions worth stealing

These are real, from this box's `.bashrc`.

**Copy from the server to your laptop's clipboard, over plain SSH:**

```bash
clip() {
    local encoded

    if [ -n "$1" ] && [ -f "$1" ]; then
        encoded=$(base64 -w 0 "$1")
    else
        encoded=$(base64 -w 0)
    fi

    # Send via OSC 52, checking for tmux environment
    if [ -n "$TMUX" ]; then
        printf "\ePtmux;\e\e]52;c;%s\a\e\\" "$encoded"
    else
        printf "\e]52;c;%s\a" "$encoded"
    fi
}
```

Usage: `cat ~/.ssh/id_ed25519.pub | clip`, then paste into GitHub. Or
`journalctl -u caddy -n 50 | clip` and paste into a chat.

How it works: OSC 52 is a terminal escape sequence that asks the *terminal emulator* to
set the system clipboard. The escape travels back over your SSH connection like any other
output, so the program runs on the server and the clipboard that changes is the one on
your laptop. No X11 forwarding, no `xclip`, no scp round trip. The `$TMUX` branch wraps
it in tmux's passthrough form so the sequence survives the multiplexer.

Your terminal has to allow it. kitty, WezTerm, iTerm2, and Windows Terminal all do; some
need a setting enabled.

**Turn a local folder into a GitHub repo in one command:**

```bash
ghnew() {
    local name="${1:-$(basename "$PWD")}"

    if [ ! -d ".git" ]; then
        git init
        git add .
        git commit -m "feat: initial commit"
    fi

    gh repo create "$name" --private --source . --remote origin --push
}
```

**Fuzzy-search the system dictionary**, which is more useful than it sounds when you are
naming things:

```bash
dict-find() {
    if [ -z "$1" ]; then
        fzf < /usr/share/dict/words
    else
        look -f "$1" /usr/share/dict/words | fzf --query="$1"
    fi
}
```

## tmux, or lose your work

Every SSH session dies when your connection drops. Anything running in it dies with it.

```bash
sudo apt install tmux
tmux new -s work      # start
# Ctrl-b then d       # detach, leave everything running
tmux attach -t work   # come back, from anywhere
```

This is not the solution for running your *service* (chapters 04 and 05 cover that
properly). It is the solution for running a build, or a migration, or anything that takes
longer than your wifi is stable.

## One warning about installers

Some tools append to your `.bashrc` without asking. This box has a block from a Kubernetes
installer wrapped in `# GAZELLE_START` and `#GAZELLE_END` comments, containing an alias
pointing at a directory that was deleted months ago.

Two lessons. Good installers mark their territory with delimiter comments so you can find
and remove the block later. And shell config rots: read your own `.bashrc` occasionally
and delete what no longer means anything.

Next: getting a runtime onto the box, and the PATH problem that follows it everywhere.
