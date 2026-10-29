

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
```

Env vars:

- `CCD_PROFILE` - config profile, stored in `~/.local/share/$CCD_PROFILE` (default `claude-agent`)
- `CCD_MEM` - memory limit (default `8g`)
- `CCD_CPUS` - cpu limit (default `4`)

Arguments are passed to `claude`, e.g. `ccd --resume` or `ccd -p "..." | less` (a TTY is only allocated when attached to a terminal).

@TODO - restricted network/egress
#    --network agent-net -e HTTPS_PROXY=http://egress-proxy:3128 \
