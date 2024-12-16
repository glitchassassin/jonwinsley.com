---
layout: post
title: "Terminal Bell for Long-Running Commands"
date: 2024-12-16 14:29:00
author: Jon Winsley
summary: Zsh snippet to sound a BELL character when a long-running command finishes
---

I work on Windows/WSL2 with multiple desktops, and sometimes when I have long-running tests I'll switch screens and completely lose track of where I was.

No longer!

Visual Studio Code (or Cursor) can play an audible cue from a terminal BELL character:

![VSCode Settings -> search for Bell -> set sound to on](/assets/vscode-bell-settings.png)

Then, in my `.zshrc`, I've added a couple zsh hooks that tracks when a command is started, and sends a bell when it's finished, if it took longer than 1 second:

```zsh
autoload -Uz add-zsh-hook

function notify_long_running_commands() {
  local stop=$SECONDS
  local elapsed=$(( stop - start ))
  if (( elapsed > 1 )); then
    echo -e "\a"
  fi
}

function track_start_time() {
  start=$SECONDS
}

add-zsh-hook preexec track_start_time
add-zsh-hook precmd notify_long_running_commands
```

Adjust your timeout to suit your attention span.
