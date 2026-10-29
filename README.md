

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

## claude code updates

The claude code version is pinned in `dockerfiles/claude-code/Dockerfile` (`ARG CLAUDE_VERSION`). It sits at the top, so a bump rebuilds every layer, including an `apt-get upgrade`.

`ccd-update` checks for a new release (one small request to the release endpoint, compared with the pinned version and the built image's `claude-code.version` label). Exits 0 if up to date, 1 if an update is available.
`ccd-update -u` bumps the pinned version and rebuilds `localhost/claude-agent:latest`.

@TODO - restricted network/egress
#    --network agent-net -e HTTPS_PROXY=http://egress-proxy:3128 \
