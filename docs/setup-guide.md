# Pasting clipboard images into agents running over SSH

A practical setup and review guide for [cc-clip](https://github.com/ShunmeiCho/cc-clip), written for whoever
runs and approves the machines. Every command and observation below was run against a real macOS laptop and an
Ubuntu 24.04 VM. Where something contradicts cc-clip's own documentation, that is called out.

cc-clip is not our software. This page exists because we evaluated building an alternative, decided cc-clip was
the better answer, and wanted the setup and the security review written down in one place.

## What problem this solves

Coding agents (Claude Code, opencode) accept pasted screenshots. They read the clipboard by running `xclip` or
`wl-paste` on the machine they are running on. When the agent runs on a server over SSH, that machine is the
server, which has no access to your laptop's clipboard, so pasting a screenshot does nothing.

This matters because it is the one thing blocking agents from moving off laptops onto servers. Once they move,
the laptop is free to sleep.

## How it works

1. A small daemon runs on the laptop and serves the current clipboard image over HTTP on `127.0.0.1:18339`.
2. Each SSH connection carries `RemoteForward 18339 127.0.0.1:18339`, so the remote host gets its own
   `127.0.0.1:18339` that tunnels back to the laptop.
3. On the remote host, a shim named `xclip` sits early in `PATH`. When an agent runs `xclip`, the shim fetches
   the image through the tunnel instead. Requests it does not serve pass through to the real binary.

No new ports are opened to the network, nothing listens on a public interface, and there is no new
authentication system: it rides the SSH connection that already exists.

## Prerequisites

**Laptop (macOS 13 or later):**

- Homebrew. cc-clip requires `pngpaste` and installs it automatically with Homebrew during setup.
- A named `Host` entry in `~/.ssh/config` for each server. `cc-clip setup` edits that entry and will not work
  against a bare hostname.

**Each remote host:**

- Linux on amd64 or arm64.
- `curl` and `bash`. The shim is a bash script that uses `curl`.
- **Not** `xclip` or `wl-paste`. cc-clip's README lists one of them as required. In practice the shim answers
  on its own, and pasting worked on a host that had neither installed. The real binary is only needed so that
  unrelated clipboard calls can pass through. Worth knowing before anyone installs X11 packages on a server
  that does not need them.

## Setting it up

### Once, on the laptop

```sh
curl -fsSL https://raw.githubusercontent.com/ShunmeiCho/cc-clip/main/scripts/install.sh | sh
cc-clip --version
```

The installer downloads a release tarball, verifies it against `checksums.txt`, installs to `~/.local/bin`, and
on macOS clears the quarantine attribute and ad-hoc signs the binary so Gatekeeper allows it. It uses no `sudo`
and changes nothing system-wide. Reading it before running it is reasonable:

```sh
curl -fsSL https://raw.githubusercontent.com/ShunmeiCho/cc-clip/main/scripts/install.sh -o /tmp/cc-clip-install.sh
less /tmp/cc-clip-install.sh && sh /tmp/cc-clip-install.sh
```

### Once per server

Make sure the host has a named entry in `~/.ssh/config`:

```
Host myserver
    HostName myserver.internal
    User youruser
```

Then:

```sh
cc-clip setup myserver
```

That single command installs the remote binary and shim, syncs an access token, adds the `RemoteForward` to
your SSH config, starts the local daemon as a launchd agent, and verifies the tunnel end to end.

### Daily use

**Press Ctrl+V, not Cmd+V.**

This catches everyone. On macOS, Cmd+V is handled by the terminal application itself: it pastes the clipboard's
*text* into the session and never reaches the agent. Ctrl+V is what the agent listens for, and it passes
through tmux untouched because tmux has no default binding for it.

If you paste with Cmd+V you get the message `no image over SSH, copy the file over, paste its path`, which
looks like a transport failure but is not. It is the ordinary "no image in clipboard" message with wording the
agent uses when it detects an SSH session.

A reconnect that both attaches and carries the tunnel:

```sh
ssh -t youruser@myserver tmux new -A -s work
```

`tmux new -A -s` attaches if the session exists and creates it otherwise, so this one command covers the first
connection and every reconnection after the laptop sleeps.

## What gets installed, and where

| Machine | Path | What it is |
|---|---|---|
| Laptop | `~/.local/bin/cc-clip` | the binary |
| Laptop | `~/Library/LaunchAgents/com.cc-clip.daemon.plist` | launchd agent, listening on `127.0.0.1:18339` |
| Laptop | `~/.ssh/config` | `RemoteForward`, `ControlMaster no` and `ControlPath none` added to the host's block |
| Laptop | Homebrew `pngpaste` | installed if missing |
| Server | `~/.local/bin/cc-clip` | the binary |
| Server | `~/.local/bin/xclip` | the shim, a bash script |
| Server | `~/.bashrc` | a marked `PATH` block |
| Server | `~/.claude/settings.json` | notification hooks, rewritten |
| Server | `~/.cache/cc-clip/` | access token and session id |

Everything is per-user. Nothing is installed system-wide and `sudo` is never needed, unless you want Codex
support, which is a different mechanism (see below).

## Verifying it

```sh
cc-clip status
cc-clip doctor --host myserver
```

`doctor` runs about eighteen checks across both machines and is genuinely good. Healthy output ends with:

```
  tunnel:        [pass] port 18339 forwarded and cc-clip daemon answering
  remote-token:  [pass] token file present
  token-match:   [pass] remote token matches local
  image-probe:   [pass] clipboard has image (verified from remote)
```

To test the mechanism without involving an agent at all, run on the server exactly what the agent runs. This is
the real contract, read out of the Claude Code binary, so if these two succeed then pasting will work:

```sh
xclip -selection clipboard -t TARGETS -o | grep -E "image/(png|jpeg|jpg|gif|webp|bmp)"
xclip -selection clipboard -t image/png -o > /tmp/test.png && file /tmp/test.png
```

Expect `image/png` from the first and a valid PNG from the second. The transfer is lossless: we compared a
source image and its arrival sample by sample and found zero differing values out of 48000.

To check whether a tunnel exists at all, on the server:

```sh
ss -ltn | grep 18339
```

No output means no connection is currently carrying the forward, and pasting cannot work until you reconnect.

## Operational notes

**Multiple servers at the same time: fine.** The forwarded port lives on each server's own loopback interface,
so `server-a:18339` and `server-b:18339` are unrelated. They all tunnel back to the one laptop daemon, which
serves them from a single listener. Verified with two hosts connected simultaneously.

**Multiple agents on one server: fine.** They all run the same shim against that server's loopback port.

**Multiple SSH connections to the *same* server: this is the one sharp edge.** Whichever connection binds port
18339 owns it. A second connection prints

```
Warning: remote port forwarding failed for listen port 18339
```

which is harmless, and pasting keeps working through the first connection's tunnel because the listener is
machine-wide. But if the owning connection closes while the others stay open, pasting stops on that server
until you reconnect. Two terminals into the same box means closing the wrong one breaks pasting in the other.

**After the laptop sleeps**, reconnect. The tunnel belongs to the connection, so it dies with it. Existing tmux
sessions and running agents do not need restarting: the shim is found through `PATH` at the moment the agent
runs it, and the tunnel is machine-wide, so simply opening a new connection restores pasting inside sessions
that are already running.

**Agents that work:** Claude Code and opencode, which run `xclip` as a subprocess.

**Agents that do not:** Codex CLI and Grok, which read the X11 clipboard in-process and never run `xclip`, so
no `PATH` shim can reach them. cc-clip supports them through a different mechanism, a virtual X display (Xvfb)
on the server, which does need `sudo` to install. Cursor works but needs `DISPLAY` set in its own shell.

## Notes for security review

**What crosses the link.** Image bytes only, pulled on demand, read-only. There is no path for a server to
write to your laptop's clipboard. Text clipboard support exists as a separate endpoint and is not part of the
image paste path.

**The access control boundary is a bearer token**, not the network. The forwarded port is on the server's
loopback interface, which on a multi-user server is reachable by every local user on that machine. cc-clip's
own [SECURITY.md](https://github.com/ShunmeiCho/cc-clip/blob/main/SECURITY.md) states this plainly: "loopback
is reachable by other local users on that machine. The token is the access control boundary." The token lives
in `~/.cache/cc-clip/session.token` with restrictive permissions and a 30 day sliding expiry.

We verified the behaviour: an un-enrolled host that forwarded the port reached the daemon and was refused with
HTTP 401. The transport is open to anyone who can reach the port; the token is what stops them.

**Therefore: deploy this on single-user servers.** That is cc-clip's own recommendation ("prefer a trusted
single-user development host, a VM/container with only your account") and it is the right one. On a shared jump
host, anyone with a local account can read the token file's protection only as well as the filesystem protects
it, and can reach the port regardless.

**What it does not defend against.** Any process running as your user on that server, including an agent that
has been prompt-injected, can read your clipboard image at will while you are connected. So can root. This is
inherent to the design, not a bug: the clipboard is being deliberately exposed to that machine.

**No phone-home behaviour.** We audited this, statically and at runtime:

- Every external URL in the source is accounted for. The only network destinations are `127.0.0.1` and GitHub
  release downloads, the latter reached only by `cc-clip update` and by `setup` when it fetches the remote
  binary. Both are user-initiated. The two other URL literals are a plist DOCTYPE string and error-message text.
- No telemetry or analytics SDKs are present.
- At runtime the daemon held exactly one socket, `127.0.0.1:18339 (LISTEN)`, with zero outbound connections.

**Supply chain.** MIT licensed, source on GitHub, release binaries checksummed and verified by the installer.
Pinning a known version with `CC_CLIP_VERSION=vX.Y.Z` is supported if you would rather not track latest.

**Two defects we found**, neither security-critical but worth knowing:

- `cc-clip uninstall --help` has no help handling and attempts a real uninstall instead of printing usage.
- `setup` reported `real bin: /usr/bin/xclip` on a host where that file does not exist, so its pass-through
  path was dead. It reports the path without verifying it.
- Its SSH config editing appends to the end of `~/.ssh/config`, which puts the forward inside whatever `Host`
  block happens to be last. Check the result after running `setup`, especially if your config ends with a
  wildcard block.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `no image over SSH, copy the file over, paste its path` | usually Cmd+V instead of Ctrl+V, or no tunnel, or no image on the clipboard | press Ctrl+V; check `ss -ltn \| grep 18339` on the server |
| Nothing listening on 18339 | no current connection carries the forward | reconnect, for example `ssh -t user@host tmux new -A -s work` |
| `remote port forwarding failed for listen port 18339` | another connection to that host already owns the port | harmless; pasting works through that connection |
| Pasting stopped while a session stayed open | the connection owning the port closed | open any new connection to that host |
| `shim: [FAIL] no cc-clip shim found` | something replaced `~/.local/bin/xclip` | re-run `cc-clip setup HOST` |
| Token errors after the daemon restarted | token out of sync | `cc-clip connect HOST --token-only` |

## Removing it

In this order:

```sh
# on the server
cc-clip uninstall

# on the laptop
cc-clip uninstall --host myserver
cc-clip service uninstall
```

Then delete the `Host` block additions from `~/.ssh/config`. Check `~/.bashrc` and `~/.claude/settings.json` on
the server if you want the removal to be complete.
