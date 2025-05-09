---
date: '2025-04-23T16:28:36+03:00'
draft: false
title: 'Terminal Emulators (+ Ghostty Setup)'
categories: ["setup", "linux"]

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

## ✨ My Shell Config (with Starship)

I use `Starship` for my prompt and my shell is `zsh`:

```bash
# Wait 15 sec for starship to check files under the current working directory
scan_timeout = 15

# Disable the blank line at the start of the prompt
add_newline = false

# Replace the "❯" symbol in the prompt with "➜"
[character]
success_symbol = "[➜](bold green)"
error_symbol = "[✖](bold red)"

[username]
format = " [╭─$user]($style)@"
style_user = "bold red"
style_root = "bold red"
show_always = true

[hostname]
format = "[$hostname]($style) in "
style = "bold dimmed red"
trim_at = "-"
ssh_only = false
disabled = false

[directory]
style = "purple"
truncation_length = 0
truncate_to_repo = true
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
disabled = false

[golang]
style = "bg:#86BBD8"
format = 'via [ $symbol ($version) ]($style)'

[docker_context]
style = "bg:#06969A"
format = '[ $symbol $context ]($style)$path'

[git_commit]
disabled = false
commit_hash_length = 8

[kubernetes]
symbol = "⎈ "
```

## TODOs
- [ ] Read history of computer terminals 

## Useful Reads
- [Serial Terminal Basics](https://learn.sparkfun.com/tutorials/terminal-basics/all#res)
- [Baeldung](https://www.baeldung.com/linux/terminal-shell-tty-vs-console)
