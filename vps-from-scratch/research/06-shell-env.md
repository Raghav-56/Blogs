# Research: shell environment, SSH, git

## The three-file trap

Three files, and which ones run depends on how the shell started.

| File | Read when |
|---|---|
| `~/.profile` | login shell, **only if `~/.bash_profile` and `~/.bash_login` do not exist** |
| `~/.bash_profile` | login shell (SSH session, `su -`) |
| `~/.bashrc` | interactive non-login shell (new terminal, `tmux` pane) |
| *(none of them)* | **non-interactive**: systemd, cron, a process manager, `ssh host "cmd"` |

On this box `~/.bash_profile` exists:

```bash
source ~/.bashrc
export KUBECONFIG=/home/ubuntu/.kube/config

# bun
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
```

Consequences, all real here:

- Because `~/.bash_profile` exists, **`~/.profile` is never read for login shells**. Its
  `$HOME/bin` PATH logic and its `. "$HOME/.cargo/env"` line are dead code. `.local/bin`
  is on PATH only because `.bashrc` exports it independently.
- **bun is on PATH only via `.bash_profile`**, so only in login shells. This is exactly
  why the process manager config hardcodes `/home/ubuntu/.bun/bin/bun` and why the CI
  deploy step re-exports PATH by hand. See `07-project-scripts.md` and `09-cicd.md`.

The rule to teach: **put exports in `.bashrc` and make `.bash_profile` just
`source ~/.bashrc`**, then accept that non-interactive contexts get none of it and always
use absolute paths in service files.

## User additions to .bashrc

Lines 1 to ~125 are stock Ubuntu 24.04 boilerplate, with one in-place edit:

```bash
# Enable extended globbing (e.g., for +(foo).* patterns)
shopt -s extglob
```

Needed by the ANSI-stripping pattern in the build scripts. Everything from here down is
appended:

```bash
# Starship
eval "$(starship init bash)"
. "$HOME/.cargo/env"
eval "$(zoxide init bash)"

export PATH="/home/ubuntu/.local/bin:$PATH"

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

ghnew() {
    # Uses the provided argument, or defaults to the current directory name
    local name="${1:-$(basename "$PWD")}"

    # Check if the .git folder does not exist
    if [ ! -d ".git" ]; then
        git init
        git add .
        git commit -m "feat: initial commit"
    fi

    # Create the GitHub repository
    gh repo create "$name" --private --source . --remote origin --push
}

dict-find() {
    if [ -z "$1" ]; then
        fzf < /usr/share/dict/words
    else
        look -f "$1" /usr/share/dict/words | fzf --query="$1"
    fi
}

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

**`clip` is the best one to publish.** OSC 52 is a terminal escape sequence that asks the
*terminal emulator* to set the clipboard. Since the escape travels back over the SSH
connection like any other output, `cat file | clip` on the server puts the text in your
laptop's clipboard. No X11 forwarding, no `xclip`, no scp. The `$TMUX` branch wraps the
sequence in tmux's passthrough form so it survives the multiplexer. Your terminal has to
allow it (kitty, WezTerm, iTerm2, Windows Terminal all do; some need a setting).

`ghnew` turns "I have a folder, I want it on GitHub" into one command.

There is also a machine-generated block from a Kubernetes installer, delimited by
`# GAZELLE_START` / `#GAZELLE_END`, containing `alias k=kubectl` and friends. Its
`cdg` alias points to a directory that no longer exists. Two lessons: installers that
append to your `.bashrc` should use delimiter comments like that one does, and aliases
rot silently.

## Toolchain actually installed

```
~/.cargo/bin/   bat  btm  eza  fd  rg  zoxide  oxmgr  cargo-install-update
/usr/local/bin/ starship  helm  k9s  kubectl  kubectx  kubens  kustomize
~/.bun/bin/     bun  bunx
```

`nvm` is present and sourced by `.bashrc` but `~/.nvm/versions/node` does not exist, so
no Node versions are installed through it. `node` resolves to `/usr/bin/node` from apt.
The nvm sourcing is pure startup cost for nothing, a good example of shell-config drift.

## SSH config

`~/.ssh/config`:

```
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes
    # IdentitiesOnly yes
```

This is the single most useful trick in the file for a student audience. Many campus and
office networks block outbound port 22. GitHub runs the same SSH service on
`ssh.github.com:443`, which looks like HTTPS to a firewall. Four lines and `git push`
works from a network that blocks SSH.

Verify with `ssh -T git@github.com`.

## git config

```ini
[init]
	defaultBranch = main
[user]
	name = Raghav Gupta
	email = 142162663+Raghav-56@users.noreply.github.com
[credential "https://github.com"]
	helper = 
	helper = !/usr/bin/gh auth git-credential
[credential "https://gist.github.com"]
	helper = 
	helper = !/usr/bin/gh auth git-credential
[core]
	editor = nvim
```

Two things worth explaining:

- The `@users.noreply.github.com` address. Committing with your real email publishes it
  in every commit, permanently, to anyone who clones. GitHub gives you a noreply alias
  in Settings, and commits with it still count towards your contribution graph.
- The empty `helper =` lines are not a mistake. Setting a credential helper to the empty
  string **clears** the list inherited from system and global config, so the `gh` helper
  that follows is the only one. Written by `gh auth login`.

```
$ gh auth status
github.com
  ✓ Logged in to github.com account Raghav-56
  - Git operations protocol: ssh
  - Token scopes: 'admin:public_key', 'gist', 'read:org', 'repo'
```

## Session recording (multi-user box)

`/usr/local/bin/record_session.sh`, used as an SSH `ForceCommand`:

```bash
#!/bin/bash

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG="/var/log/sessions/${TIMESTAMP}_${USER}_pr.log"
TIMING="/var/log/sessions/${TIMESTAMP}_${USER}_pr.timing"

mkdir -p /var/log/sessions

if [ -n "$SSH_ORIGINAL_COMMAND" ]; then
    # Run the specific command they requested, but still record the output
    exec script -q -f "$LOG" -t "$TIMING" -c "$SSH_ORIGINAL_COMMAND"
else
    # They just ran a normal 'ssh', so give them a fully normal login shell
    exec script -q -f "$LOG" -t "$TIMING" -c "tmux new-session -A -s workspace"
fi
```

`script -t` writes a timing file, so `scriptreplay` plays the session back at original
speed. Useful for a shared box, and a nice demonstration that `script` exists. Note it
also drops everyone into a shared tmux session named `workspace`.
