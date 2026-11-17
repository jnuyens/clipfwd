# Logo brief: clipfwd

## What it is

clipfwd is an open-source command-line tool (AGPL-3.0). It lets you paste a screenshot from your
laptop into a program running on a remote server over SSH. Today, Ctrl+V with an image does nothing on
a remote machine, because the clipboard sits on the laptop. clipfwd carries the image through the
existing SSH connection, so the remote program sees an ordinary clipboard.

The main audience is developers who run AI coding agents (Claude Code, Codex, opencode) on a server
instead of on their laptop.

## Name

Always lowercase: `clipfwd`, short for "clipboard forward". It's read as "clip forward".

## The idea to get across

An image travelling from your machine to a distant terminal. Something gets handed across a gap:
clipboard, then a channel, then a terminal prompt. It should feel small, sturdy and trustworthy. It's a
utility, not an app.

## Possible directions

Pick one or two to explore:

1. A paper clip, or a clipboard clip, that bends into a forward arrow or a `>` prompt.
2. A small image tile (a photo frame with a mountain or sun) sliding through a tunnel or pipe into a
   terminal cursor `_`.
3. A lettermark built from `cf` or `c>`, where the `>` doubles as the shell prompt and the forward arrow.

## Must work

- At 16x16 and 32x32 pixels (favicon, GitHub avatar) without losing its shape.
- In a single colour, on both dark and light backgrounds. Many terminals and the GitHub dark theme are
  dark.
- As a square symbol on its own, and as a horizontal lockup with the wordmark `clipfwd`.

## Style

- Flat, geometric, few strokes. A monospaced or simple geometric typeface for the wordmark, which fits a
  terminal tool.
- One accent colour plus neutral greys. No gradients, no 3D, no glossy app-icon look.

## Avoid

- Logos or recognisable elements of other brands: Apple, Anthropic or Claude, OpenAI, GitHub, OpenSSH's
  pufferfish, Tux.
- A literal padlock or shield. Security matters, but it isn't the product.
- Fine detail that disappears at small sizes.

## Deliverables

- Symbol and horizontal lockup as SVG, plus PNG at 16, 32, 64, 128, 256 and 512 px.
- Light, dark and single-colour variants.
- A GitHub social preview image of 1280x640 px with the lockup and the line "Paste screenshots into
  remote CLI tools over SSH".
- The colour values (hex) and the typeface name, with its licence.

## Rights

The project is open source, so the logo must be free to use in the repo, the documentation, packages
(Homebrew, .deb) and talks. Please hand over the rights to the project, or license it under CC BY 4.0,
and use only fonts with an open licence such as SIL OFL.
