

## delegate cpu cpuset io memory pids cgroup controllers to user services:
`/etc/systemd/system/user@.service.d/delegate.conf`:

```
# /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=cpu cpuset io memory pids
```

```
# systemd daemon reload
❯ systemctl daemon-reload

# verify:
❯ cat /sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers
cpuset cpu io memory pids
```

## dedicated systemd slice `agents.slice`
here we define combined resource limits for all agents containers

```
# ~/.config/systemd/user/agents.slice
[Slice]
MemoryMax=10G
MemorySwapMax=0
CPUQuota=600%
CPUWeight=50
```

## wrapper scripts

`bin/ccd` runs claude code in the container for the current directory; `bin/ccdw` is the same with the `claude-agent-wedos` profile.
Symlink them into `~/bin` (or anywhere on `$PATH`):

```
ln -s "$PWD"/bin/ccd ~/bin/ccd
ln -s "$PWD"/bin/ccdw ~/bin/ccdw
ln -s "$PWD"/bin/ccd-update ~/bin/ccd-update
```

Env vars:

- `CCD_PROFILE` - config profile, stored in `~/.local/share/$CCD_PROFILE` (default `claude-agent`)
- `CCD_MEM` - memory limit (default `8g`)
- `CCD_CPUS` - cpu limit (default `4`)

Arguments are passed to `claude`, e.g. `ccd --resume` or `ccd -p "..." | less` (a TTY is only allocated when attached to a terminal).

## git config

Each profile can have its own git config in `~/.config/ccd/<profile>.gitconfig`. If the file exists, `ccd` mounts `~/.config/ccd` read-only and points `GIT_CONFIG_GLOBAL` at it, so the agent can't change its identity or config. Shared settings go to `common.gitconfig`, included from the profile files.

```
mkdir -p ~/.config/ccd
cp examples/gitconfig/*.gitconfig ~/.config/ccd/
# then edit [user] in ~/.config/ccd/<profile>.gitconfig
```

No credentials or ssh keys are mounted - push from the host.

## worktrees

`ccd -w [name]` (or `--worktree[=name]`) runs claude's native worktree mode: the agent works in `.claude/worktrees/<name>` on branch `worktree-<name>`, created from your local `HEAD` (`worktree.baseRef: head`), and Claude Code offers to clean it up on exit. Gitignored files listed in `.worktreeinclude` (`.gitignore` syntax) are copied into new worktrees.

Mounts in worktree mode:

| path | mount |
|---|---|
| `<repo>` | read-only - your files; agent can still read untracked ones (`.env`, data) |
| `<repo>/.git` | read-write - commits, branches, worktree metadata |
| `<repo>/.git/config`, `<repo>/.git/hooks` | read-only - both run code on the host |
| `<repo>/.claude/worktrees` | read-write - the agent's worktrees |

The agent also gets `--add-dir <repo>` and a short system prompt note about the read-only checkout. Without `-w`, `.git/config` and `.git/hooks` are still read-only when `ccd` runs in the repo root.

Notes:

- add `.claude/worktrees/` to your global gitignore (`~/.config/git/ignore`) so it doesn't show up as untracked
- the agent can still create and move refs (needed to commit), including your branches
- permission approvals aren't saved in worktree mode (the checkout's `.claude/settings.local.json` is read-only)
- review and merge on the host: `git log main..worktree-<name>`, `git merge worktree-<name>`, `git worktree remove .claude/worktrees/<name>`
- deleting a branch inside the container prints `warning: update of config-file failed` - harmless, `.git/config` is read-only

## status line

The image ships a powerline-style status line (`dockerfiles/claude-code/ccd-statusline`), enabled for all profiles by `/etc/claude-code/managed-settings.json`:

```
 claude-agent-1234  claude-code-container/bin   main ⇡1 ✚2 ●1 ?3  Opus 5.5  ctx 34%  5h 12%
```

- host segment: profile + `ccd` pid, colour derived from the profile name; the full container name (`podman exec`) also has the directory
- path: repo name + subpath (`repo ⎇ name/subpath` in a linked worktree), shortened path outside a repo
- git: `⇡` ahead, `⇣` behind, `✖` conflicts, `✚` staged, `●` modified, `?` untracked
- `ctx` and `5h`: yellow from 50%, red from 80%; `5h` only shows on claude.ai plans, after the first response
- segments drop from the right if the terminal is too narrow

Needs a powerline/nerd font in the host terminal, none in the image.

## claude code updates

The claude code version is pinned in `dockerfiles/claude-code/Dockerfile` (`ARG CLAUDE_VERSION`). It sits at the top, so a bump rebuilds every layer, including an `apt-get upgrade`.

`ccd-update` checks for a new release (one small request to the release endpoint, compared with the pinned version and the built image's `claude-code.version` label). Exits 0 if up to date, 1 if an update is available.
`ccd-update -u` bumps the pinned version and rebuilds `localhost/claude-agent:latest`.

---

@TODO - restricted network/egress
```
# podman ..
#    --network agent-net -e HTTPS_PROXY=http://egress-proxy:3128 \
```
