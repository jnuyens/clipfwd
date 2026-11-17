# clipfwd

Paste images from your local clipboard into CLI tools running over SSH.

> **Status: early development. Nothing here works yet.**
> The design is written down and the SSH mechanics are verified, but no code has been written.
> Watch the repo if you want to know when there is something to install.

## The problem

CLI tools that accept pasted images read the clipboard by running `xclip` or `wl-paste` on the machine they
are running on. Over SSH that machine is the remote host, which has no access to your laptop's clipboard.
So pressing Ctrl+V with a screenshot does nothing.

This bites hardest if you want to move a coding agent off your laptop. Running Claude Code, Codex or
opencode on a server means your laptop can sleep, but it also means you can no longer show the agent a
screenshot of what is going wrong.

## How it will work

1. A small server on your laptop serves the current clipboard image over a Unix socket.
2. SSH forwards that socket to the remote host with `RemoteForward`. No new protocol, no open ports,
   no extra authentication.
3. On the remote host, `xclip` and `wl-paste` shims sit early in `PATH`. When the socket is there they
   answer from your laptop's clipboard. When it is not, they hand off to the real binary.

The tool doing the pasting sees an ordinary clipboard and needs no changes.

## Planned scope for the first release

- macOS as the clipboard source, Linux (Debian/Ubuntu) as the remote host
- One static binary that acts as the server, the shims and the installer
- `clipfwd push <host>` to set up a new remote host in one command
- `clipfwd doctor` to explain what is broken when pasting does not work
- Homebrew tap, `.deb`, and signed release builds for darwin and linux on amd64 and arm64

Linux as the clipboard source, text clipboard support and Windows come later.

## Security

Read this before enabling the forward.

While you are connected, any process running as your user on the remote host can read your clipboard
image. So can root. Do not enable the forward for hosts you do not trust, and prefer listing hosts
explicitly over using `Host *`.

Design choices that follow from that:

- The socket lives under your home directory, inside a directory with mode 0700, on both ends. Socket
  file permissions are not honored on every operating system, but directory permissions always are.
  A socket in `/tmp` would also be squattable by another local user, which would let them feed content
  of their choosing to your agent.
- Images only. Text clipboard support is deliberately absent, and will be opt-in when it arrives,
  because text clipboards routinely hold passwords.
- The server only reads. There is no way for a remote host to change your clipboard.
- Requests have a size cap and a short timeout.

## License

[AGPL-3.0](LICENSE)
