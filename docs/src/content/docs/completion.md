---
title: Command-line completion
sidebar:
  order: 4
---

Tanka supports CLI completion for `bash`, `zsh` and `fish`.

```bash
# Install
tk complete

# Uninstall
tk complete --remove
```

As tanka is its own completion handler, it needs to hook into your shell's
configuration file (`.bashrc`, etc).

When using other shells than `bash`, Tanka relies on a _Bash compatibility
mode_. It enables this automatically when installing, but please make sure no
other completion (e.g. OhMyZsh) interferes with this, or your completion might
not work properly.  
It sometimes depends on the order the completions are being loaded, so try
putting Tanka before or after the others.
