

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
ln -s "$PWD"/bin/ccd-proxy ~/bin/ccd-proxy
```

Env vars:

- `CCD_PROFILE` - config profile, stored in `~/.local/share/$CCD_PROFILE` (default `claude-agent`)
- `CCD_MEM` - memory limit (default `8g`)
- `CCD_CPUS` - cpu limit (default `4`)
- `CCD_TMP_DAYS` - delete files in the agent's temp dir older than this many days, `0` keeps everything (default `14`, see [temp files](#temp-files))
- `CCD_NET` - `open` skips the egress proxy for this run (default: restricted, see [network access](#network-access))

Arguments are passed to `claude`, e.g. `ccd --resume` or `ccd -p "..." | less` (a TTY is only allocated when attached to a terminal).

## temp files

`/tmp` is a 2 GB tmpfs, gone when the container exits. Claude Code's own temp dir (scratchpads, background task output) is moved by `CLAUDE_CODE_TMPDIR=/tmp/ccd-<user>-<profile>/<repo>-<hash>` onto a host mount, `~/.cache/ccd/<profile>/tmp/<repo>-<hash>`, so files the agent leaves there survive restarts and `--resume`. `ccd` also creates the host symlink `/tmp/ccd-<user>-<profile> -> ~/.cache/ccd/<profile>/tmp`, so paths the agent prints work as-is on the host. `<repo>-<hash>` is the git repo (main checkout) `ccd` runs in, or `$PWD` outside a repo; a container only gets its own one mounted:

```
/tmp/ccd-<user>-<profile>/<repo>-<hash>/claude-<uid>/<encoded-project-path>/<session-id>/scratchpad/
```

`XDG_RUNTIME_DIR` is a per-container tmpfs at `/run/user/<uid>`, so Claude Code's messaging sockets (`cc-socks/<pid>.sock`, pid 1 in every container) don't end up in the shared mount.

If `/tmp/ccd-<user>-<profile>` already exists on the host and isn't that symlink, `ccd` warns and leaves it alone.

On each start `ccd` deletes everything (files, sockets, directories) older than `CCD_TMP_DAYS` (default 14) by mtime, from any session; a directory goes only once it is empty, so recent directories stay even when empty. That temp dir isn't limited by the tmpfs size: put `~/.cache/ccd` on its own volume if you want a cap.

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
`ccd-update -u` bumps the pinned version and rebuilds `localhost/claude-agent:latest` and `localhost/ccd-proxy:latest`. Running proxies keep the old image until `ccd-proxy restart`.

## network access

Agents only reach allowlisted domains, through a squid proxy:

```
agent -- ccd-net-<profile> (internal) -- ccd-proxy-<profile> -- ccd-egress-<profile> -- internet
```

`ccd-net-<profile>` is a podman `--internal` network: no route out and no external DNS. The proxy env vars are only a convenience, an agent that unsets them gets no connectivity. Plain `git+ssh` doesn't work either (push from the host).

Build the proxy image once, `ccd` starts the proxy on first use and leaves it running. If the proxy can't start, `ccd` stops instead of running with an open network.

```
podman build -t localhost/ccd-proxy:latest dockerfiles/egress-proxy
ccd-proxy check
```

Allowlists are on the host, read-only for the agent:

- `~/.config/ccd/common.allowlist` - all profiles, created from `examples/allowlist/common.allowlist` on first start: Claude Code, PyPI, npm, GitHub, Debian, Terraform, Helm
- `~/.config/ccd/<profile>.allowlist` - additions for one profile, created empty

```
example.com    # only this host
.example.com   # this host and all subdomains
```

Don't list a host that a `.domain` entry already covers. Only ports 80 and 443 are allowed, and names that resolve to private or loopback addresses are blocked.

`ccd-proxy` works on the proxy of `CCD_PROFILE`:

- `ccd-proxy reload` - apply allowlist changes without restarting agents; an invalid list is reported and the proxy keeps running
- `ccd-proxy log` - follow the access log, blocked requests show as `TCP_DENIED`
- `ccd-proxy restart` - recreate the proxy from the current image, cuts running agents' connections
- `ccd-proxy check` - test the restrictions from a throwaway agent container

`CCD_NET=open ccd` runs a single agent on podman's default network, without the proxy.

Limitations:

- filtering is by domain only, without TLS interception, so an allowed domain is allowed entirely. Data can still leave through allowed hosts: `api.anthropic.com` with someone else's API key, GitHub or npm if a token is readable in the project
- the proxy doesn't compare TLS SNI with the requested host, so domain fronting through a CDN that serves an allowed domain is possible in principle
- agents of the same profile share a network and can reach each other; different profiles can't
- WebFetch fails for domains not on the list; WebSearch runs on Anthropic's side and keeps working
- not listed on purpose: `storage.googleapis.com` (anyone's bucket, only used for `/plugin` metadata) and the optional Datadog telemetry hosts
