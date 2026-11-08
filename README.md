# claude-code-container

Run [Claude Code](https://claude.com/claude-code) in a rootless podman container, so you can give the agent a lot of autonomy without giving it your machine.

`ccd` starts a throwaway container for the current directory and runs `claude` in it. The agent sees:

- the current directory, read-write; when that's the repo root, `.git/config` and `.git/hooks` are read-only
- its own claude config dir per [profile](#profiles), not your `~/.claude`
- none of your ssh keys, credentials or the rest of your home dir
- only allowlisted domains on the network, through a per-profile [egress proxy](#network-access)

The container runs as your uid (rootless, `--userns=keep-id`) with all capabilities dropped, `no-new-privileges`, and memory, cpu and pid limits inside a shared [`agents.slice`](#2-systemd-slice-agentsslice). The image (Debian trixie) has git, python, node, playwright with chromium, terraform and helm.

Requirements: Linux with systemd, rootless podman, bash and git on the host.

## what it doesn't protect

- **the project is untrusted after a session.** Anything the agent writes can run code on the host when you use it: build files, `package.json` scripts, `.envrc`, the project's `.claude/settings.json`, editor configs. Git is affected too, even with `.git/config` and `.git/hooks` read-only: a nested `.git` in a subdir, `.git/commondir`, submodule gitdirs under `.git/modules` or a worktree's `.git` file can point git at a config the agent wrote (e.g. `core.fsmonitor`). Plain `git status` in such a dir runs it, and so does a shell prompt that shows git status. Review the diff before building or running anything, and check for stray `.git` entries (`find . -name .git`, `ls .git`).
- **projects of one profile share the config dir.** The agent can read other projects' transcripts and the profile's claude login (`.credentials.json`), and change settings, hooks and MCP servers that apply to the profile's later sessions. Use separate [profiles](#profiles) for projects that must not see each other.
- **secrets in the project are readable**, `.env` files or a token in `.git/config` remote URLs included, and can leave through allowed domains (see [network limitations](#network-access)).
- `~/.config/ccd` is mounted read-only as a whole when the profile has a gitconfig, so the agent sees all profiles' gitconfigs and allowlists.
- the git identity isn't enforced: the agent can't change the config, but can commit with `git -c user.email=...`.
- disk usage in the project and the [temp dir](#temp-files) isn't limited.

## quick start

```
cd ~/git/myproject
ccd                     # interactive session in the container
ccd --resume            # arguments go to claude
ccd -w feature-x        # work in a git worktree, your checkout stays read-only
ccd -p "fix the tests" | less
```

Review the changes and push from the host. The agent can commit, but has no ssh keys or git credentials to push with (unless the project contains a token).

## setup

### 1. delegate cgroup controllers to user services

Needed for the memory and cpu limits of rootless containers.

```
# /etc/systemd/system/user@.service.d/delegate.conf
[Service]
Delegate=cpu cpuset io memory pids
```

```
❯ sudo systemctl daemon-reload

# verify (after logging in again):
❯ cat /sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers
cpuset cpu io memory pids
```

### 2. systemd slice `agents.slice`

Combined resource limits for all agent containers and proxies:

```
# ~/.config/systemd/user/agents.slice
[Slice]
MemoryMax=10G
MemorySwapMax=0
CPUQuota=600%
CPUWeight=50
```

```
❯ systemctl --user daemon-reload
```

### 3. build the images

```
podman build -t localhost/claude-agent:latest dockerfiles/claude-code
podman build -t localhost/ccd-proxy:latest dockerfiles/egress-proxy
```

Later updates are handled by [`ccd-update`](#claude-code-updates).

### 4. install the scripts

Symlink them into `~/bin` (or anywhere on `$PATH`):

```
ln -s "$PWD"/bin/ccd ~/bin/ccd
ln -s "$PWD"/bin/ccd-update ~/bin/ccd-update
ln -s "$PWD"/bin/ccd-proxy ~/bin/ccd-proxy
```

On the first `ccd` run claude asks you to log in; the login is kept in the profile's config dir. Optionally set up a [git identity](#git-config) for the agent's commits.

## ccd

`ccd [claude args]` runs claude code in the container for the current directory. Arguments are passed to `claude`; a TTY is only allocated when attached to a terminal, so `ccd -p "..." | less` works.

Env vars:

- `CCD_PROFILE` - config profile, stored in `~/.local/share/$CCD_PROFILE` (default `claude-agent`, see [profiles](#profiles))
- `CCD_MEM` - memory limit (default `8g`)
- `CCD_CPUS` - cpu limit (default `4`)
- `CCD_TMP_DAYS` - delete a project's temp dir when no `ccd` has started for it in this many days, `0` keeps everything (default `14`, see [temp files](#temp-files))
- `CCD_NET` - `open` skips the egress proxy for this run (default: restricted, see [network access](#network-access))

## profiles

A profile is a separate agent identity with its own claude login and settings (`~/.local/share/<profile>`), git config, allowlist, proxy, network and temp dirs. Use one per account or client to keep them apart. Names may contain letters, digits, `_`, `.` and `-`.

To add a profile, e.g. `claude-agent-work`, put a wrapper on your `$PATH`:

```
#!/bin/bash
# ~/bin/ccd-work
CCD_PROFILE=claude-agent-work exec ccd "$@"
```

Optionally give it its own git identity (see [git config](#git-config)) and allowlist additions:

```
cp examples/gitconfig/claude-agent.gitconfig ~/.config/ccd/claude-agent-work.gitconfig
# edit [user] in it
# add domains to ~/.config/ccd/claude-agent-work.allowlist (created empty on first start)
```

On first start the profile gets its own config dir and proxy, and claude asks you to log in. `ccd-proxy` acts on `CCD_PROFILE`, e.g. `CCD_PROFILE=claude-agent-work ccd-proxy log`.

## temp files

`/tmp` is a 2 GB tmpfs, gone when the container exits. Claude Code's own temp dir (scratchpads, background task output) is moved by `CLAUDE_CODE_TMPDIR=/tmp/ccd-<user>-<profile>/<repo>-<hash>` onto a host mount, `~/.cache/ccd/<profile>/tmp/<repo>-<hash>`, so files the agent leaves there survive restarts and `--resume`. `ccd` also creates the host symlink `/tmp/ccd-<user>-<profile> -> ~/.cache/ccd/<profile>/tmp`, so paths the agent prints work as-is on the host. `<repo>-<hash>` is the git repo (main checkout) `ccd` runs in, or `$PWD` outside a repo; a container only gets its own one mounted:

```
/tmp/ccd-<user>-<profile>/<repo>-<hash>/claude-<uid>/<encoded-project-path>/<session-id>/scratchpad/
```

`XDG_RUNTIME_DIR` is a per-container tmpfs at `/run/user/<uid>`, so Claude Code's messaging sockets (`cc-socks/<pid>.sock`, pid 1 in every container) don't end up in the shared mount.

If `/tmp/ccd-<user>-<profile>` already exists on the host and isn't that symlink, `ccd` warns and leaves it alone.

Each start touches `~/.cache/ccd/<profile>/tmp/<repo>-<hash>.stamp` (next to the mounted dir, so the agent can't touch it) and removes the whole `<repo>-<hash>` tree of every project whose stamp is older than `CCD_TMP_DAYS` (default 14), without walking the trees. A container running longer than that without another `ccd` start in the same project can lose its temp dir under it. Dirs without a stamp (from before stamps existed) get one and are counted from then. That temp dir isn't limited by the tmpfs size: put `~/.cache/ccd` on its own volume if you want a cap.

## git config

Each profile can have its own git config in `~/.config/ccd/<profile>.gitconfig`. If the file exists, `ccd` mounts `~/.config/ccd` read-only and points `GIT_CONFIG_GLOBAL` at it, so the agent can't change the file. It can still override the identity per command (`git -c`, `GIT_AUTHOR_*`), so this sets a default, not an enforced identity. Shared settings go to `common.gitconfig`, included from the profile files.

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

The agent also gets `--add-dir <repo>` and a short system prompt note about the read-only checkout. Without `-w`, `.git/config` and `.git/hooks` are still read-only when `ccd` runs in the repo root; in a subdir the repo's `.git` isn't mounted at all, so git doesn't work there. Read-only config and hooks don't make `.git` safe, see [what it doesn't protect](#what-it-doesnt-protect).

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

`ccd` starts the proxy on first use and leaves it running. If the proxy can't start, `ccd` stops instead of running with an open network. `ccd-proxy check` tests the restrictions.

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

`CCD_NET=open ccd` runs a single agent on podman's default network, without the proxy: full internet access, and with pasta the host itself is reachable at `host.containers.internal`.

Limitations:

- filtering is by domain only, without TLS interception, so an allowed domain is allowed entirely. Data can still leave through allowed hosts: `api.anthropic.com` with someone else's API key, GitHub or npm if a token is readable in the project
- the proxy doesn't compare TLS SNI with the requested host, so domain fronting through a CDN that serves an allowed domain is possible in principle
- agents of the same profile share a network and can reach each other; different profiles can't
- WebFetch fails for domains not on the list; WebSearch runs on Anthropic's side and keeps working
- not listed on purpose: `storage.googleapis.com` (anyone's bucket, only used for `/plugin` metadata) and the optional Datadog telemetry hosts

## License

MIT, see [LICENSE](LICENSE).
