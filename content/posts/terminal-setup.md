---
date: '2025-04-23T16:28:36+03:00'
draft: false
title: 'Terminal Emulators (+ Ghostty Setup)'
tags: ["terminal", "linux", "console", "unix", "shell"]

---

Lately, I’ve been filling in some of the gaps in my software engineering knowledge. One random question that I had was this little message in my Terminal:

```bash
Last login: Sat Apr 19 00:15:16 on ttys001
```

What does *ttys001* even mean?

---

## 🖥️ Terminal, Console, TTY — What's the Difference?

That question led me to explore how terminals actually work. I had always used the default Terminal app on macOS without giving it much thought. Now I know that on modern-day computers, we usually use the word terminal to refer to software programs known as terminal emulators:

- **Shell**: A shell is a program that acts as a command-line interface  etween you and the operating system. Shells are applications that use the kernel API in just the same way as it is used by other application programs. A shell manages the user–system interaction by prompting users for input, interpreting their input, and then handling output from the underlying operating system.
- **Terminal**: Originally, a physical device that was connected to a computer to give access to a shell. In those days, you would have a large mainframe computer with many "dumb terminals" connected to it - they had no computing abilities themselves, they just sent keystrokes to the mainframe and displayed the text that was returned. These days, "terminal" usually refers to a terminal emulator program, which provides the window that displays the shell and allows you to interact with it.
- **Console**: The physical device or main terminal (originally tied to mainframes). A console is a special kind of terminal that is directly connected to the mainframe and used for maintenance tasks.
- **TTY**: A legacy term and acronym for the  `Teletypewriter` device that still shows up in Unix systems to refer to terminal devices. In the Linux world, tty refers to an interface that enables access to the terminal.
- **Command Line**: A terminal is not a command prompt, though the two are somewhat similar. In Mac OS, the command prompt is even called Terminal. 

```bash
//===========================\\
||          Terminal         ||
||             |-----------| ||
|| Keyboard--->|   Input   |-++->|---|   |-------|
||             |-----------| ||  |tty|<=>| shell |       
||         |---------|<------++--|---|   |-------|
|| Print<--|  Output |       ||
||         |---------|       ||
||                           ||
\\===========================//
```
Source: [Reddit](https://www.reddit.com/r/programming/comments/41u5hw/comment/cz5ejh6/)

---

## Enter: Ghostty

After browsing terminal emulators available in 2025, I decided to switch to **Ghostty**. I checked out other great ones like **Kitty**, **wezterm**, and **alacritty**. I also looked into **Warp**, but requiring a login to collect telemetry made it a no-go for me.

I chose Ghostty because:

- It’s fast — significantly faster than the stock Terminal app (some benchmarks say *10x* faster).
- It’s standards-compliant.
- It has a bunch of quality-of-life features, like themes, tab/split management, and Kitty graphics protocol support.

I’m still new to terminal emulators, so honestly, any of the modern ones would’ve blown my mind. But the Ghostty experience, especially how easy it is to apply pretty themes, see the terminal inspector - won me. Other features I am now using are windows, tabs, and splits, whereas I used to go have different tabs only. Also the Kitty graphics protocol. 

---

## My Ghostty Config

```bash
shell-integration-features = no-cursor, sudo
cursor-style = block
cursor-color = #f5e0dc
cursor-text = #cdd6f4
cursor-opacity = 0.75
adjust-cell-height = 15%

quick-terminal-position = left

clipboard-read = allow
clipboard-write = allow
copy-on-select = clipboard

theme = catppuccin-mocha
bold-is-bright = true
background = #1e1e2e
foreground = #cdd6f4
selection-background = #585b70
selection-foreground = #cdd6f4

font-size = 14.5
font-family = SFMono Nerd Font

window-save-state = always
window-padding-balance = true

# Keybindings
keybind = cmd+s>r=reload_config
keybind = cmd+s>x=close_surface

# Splits
keybind = cmd+s>\=new_split:right
keybind = cmd+s>-=new_split:down

# Tab switch
keybind = cmd+s>1=goto_tab:1
keybind = cmd+s>2=goto_tab:2
keybind = cmd+s>3=goto_tab:3
keybind = cmd+s>4=goto_tab:4
keybind = cmd+s>5=goto_tab:5
keybind = cmd+s>6=goto_tab:6
keybind = cmd+s>7=goto_tab:7
keybind = cmd+s>8=goto_tab:8
keybind = cmd+s>9=goto_tab:9

# Navigate splits
keybind = cmd+s>j=goto_split:bottom
keybind = cmd+s>k=goto_split:top
keybind = cmd+s>h=goto_split:left
keybind = cmd+s>l=goto_split:right

keybind = cmd+s>e=equalize_splits
keybind = cmd+s>z=toggle_split_zoom
```

## Fonts

```bash
ls ~/Library/Fonts

# Output
SFMono Bold Italic Nerd Font Complete
SFMono Bold Nerd Font Complete
SFMono Heavy Italic Nerd Font Complete
SFMono Heavy Nerd Font Complete
SFMono Light Italic Nerd Font Complete
SFMono Light Nerd Font Complete
SFMono Medium Italic Nerd Font Complete
SFMono Medium Nerd Font Complete
SFMono Regular Italic Nerd Font Complete
SFMono Regular Nerd Font Complete
SFMono Semibold Italic Nerd Font Complete
SFMono Semibold Nerd Font Complete
```

## ✨ My Shell Config (with Starship)

I use `Starship` for my prompt and my shell is `zsh`:

```toml
# Wait 15 sec for starship to check files under the current working directory
scan_timeout = 15

# Disable the blank line at the start of the prompt
add_newline = false

[line_break]
disabled = false

[time]
disabled = false
format = "[$time]($style) "
time_format = "[%T]"  # This gives you HH:MM:SS (24-hour format)
# style = "bold yellow"
style = "#939594"

# Replace the "❯" symbol in the prompt with "➜"
[character]
# The "success_symbol" segment is being set to "➜" with the color "bold green"
# use_symbol_for_status = true
success_symbol = "[➜](bold green)"
error_symbol = "[✖](bold red)"

[username]
format = " [╭─$user]($style)@"
style_user = "bold red"
style_root = "bold red"
show_always = false

[hostname]
format = "[$hostname]($style) in "
style = "bold dimmed red"
trim_at = "-"
ssh_only = false
disabled = true

[directory]
style = "purple"
truncation_length = 8
truncate_to_repo = false
truncation_symbol = "repo: "

[git_status]
style = "white"
ahead = "⇡${count}"
diverged = "⇕⇡${ahead_count}⇣${behind_count}"
behind = "⇣${count}"
deleted = "x"
staged = "[+](green)"
untracked = "[?](red)"
modified = "[*](yellow)"

[cmd_duration]
min_time = 1
format = "took [$duration]($style)"
disabled = true

[aws]
symbol = "\uE32D  " # requires Nerd Font, AWS icon

[gcloud]
disabled = true

[git_branch]
symbol = "\uF418 "  # requires Nerd Font, Git branch

[golang]
symbol = "\uE627 "  # requires Nerd Font, Go icon
format = "via [$symbol ($version) ]($style)"

[memory_usage]
symbol = "\uF85A "  # requires Nerd Font, Memory

[docker_context]
symbol = "\uF308 "  # requires Nerd Font, Docker
style = "bg:#06969A"
format = '[ $symbol $context ]($style)$path'

[git_commit]
disabled = false
commit_hash_length = 8

[kubernetes]
format = '[⛵ ($cluster) \($namespace\)](bold green) '
disabled = false

[package]
symbol = "\uF896"   # requires Nerd Font, Package
```

## Useful Reads
- [Serial Terminal Basics](https://learn.sparkfun.com/tutorials/terminal-basics/all#res)
- [Baeldung](https://www.baeldung.com/linux/terminal-shell-tty-vs-console)
- [A Guide to the Terminal, Console, and Shell](https://thevaluable.dev/guide-terminal-shell-console/)
