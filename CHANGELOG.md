# Changelog

All notable changes to this project are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.1.0] - 2026-11-08

First release.

### Added

- `ccd`: runs Claude Code in a rootless podman container for the current directory
  - all capabilities dropped, `no-new-privileges`, memory, cpu and pid limits in a shared `agents.slice`
  - per-profile claude config dir (`CCD_PROFILE`), separate from `~/.claude`
  - `.git/config` and `.git/hooks` read-only in the repo root
  - worktree mode (`ccd -w`) on top of `claude --worktree`, the checkout itself read-only
  - per-profile read-only git config (`~/.config/ccd/<profile>.gitconfig`), with examples
  - Claude Code's temp dir on a host mount at the same path as in the container, per project, cleaned up after `CCD_TMP_DAYS`
  - per-container tmpfs `XDG_RUNTIME_DIR`
- per-profile egress proxy (squid) on an internal network: allowlisted domains only, ports 80/443, private and loopback addresses blocked
- `ccd-proxy`: start, reload, log, restart and check the proxy
- `ccd-update`: check for a new Claude Code release, bump the pinned version and rebuild the images
- agent image (Debian trixie) with git, python, node, playwright with chromium, terraform and helm, and a powerline status line
- README with setup steps and a list of what `ccd` doesn't protect
- MIT license

[0.1.0]: https://github.com/mrakopes/claude-code-container/releases/tag/v0.1.0
