

## deleagte cpu cpuset io memory pids cgroup controllers to user services:
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

## dedicated systemd slice `agenst.slice`
here we define combined resource limits for all agents containers

```
# ~/.config/systemd/user/agents.slice
[Slice]
MemoryMax=10G
MemorySwapMax=0
CPUQuota=600%
CPUWeight=50
```

## shell wrapper

```
ccd() {
  podman run --rm -it \
    --userns=keep-id --cap-drop=ALL --security-opt no-new-privileges \
    --cgroup-parent=agents.slice \
    --memory="${CCD_MEM:-8g}" --memory-swap="${CCD_MEM:-8g}" \
    --cpus="${CCD_CPUS:-4}" --cpu-shares=512 \
    --pids-limit=2048 \
    --tmpfs /tmp:rw,size=2g,mode=1777 \
    --ulimit nofile=8192:8192 \
    -v "$PWD":"$PWD" -w "$PWD" \
    -v ~/.local/share/claude-agent:/home/agent/.claude \
    -e CLAUDE_CONFIG_DIR=/home/agent/.claude \
    --name "claude-$(basename "$PWD")" \
    localhost/claude-agent:latest claude "$@"
}
```

@TODO - restricted network/egress
#    --network agent-net -e HTTPS_PROXY=http://egress-proxy:3128 \




