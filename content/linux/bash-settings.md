---
title: "Bash Settings"
date: 2023-10-03T10:40:17Z
draft: False
---

Some nice bash settings.

{{< toc >}}

## Install some prerequisites
```
apt install ccze
```

## Better bash prompt
Set this for users.
```plain
PS1="${debian_chroot:+($debian_chroot)}\[\033[38;5;231m\][\[$(tput sgr0)\]\[\033[38;5;122m\]\t\[$(tput sgr0)\]\[\033[38;5;231m\]]\[$(tput sgr0)\] \[$(tput sgr0)\]\[\033[38;5;49m\]\u\[$(tput sgr0)\]\[\033[38;5;220m\]@\[$(tput sgr0)\]\[\033[38;5;45m\]\h\[$(tput sgr0)\]\[\033[38;5;231m\]:\w\[$(tput sgr0)\] \[$(tput sgr0)\]\[\033[38;5;155m\]\\$\[$(tput sgr0)\] \[$(tput sgr0)\]"
```

Set this for root.
```plain
PS1="${debian_chroot:+($debian_chroot)}\[\033[38;5;231m\][\[$(tput sgr0)\]\[\033[38;5;122m\]\t\[$(tput sgr0)\]\[\033[38;5;231m\]]\[$(tput sgr0)\] \[$(tput sgr0)\]\[\033[38;5;1m\]\u\[$(tput sgr0)\]\[\033[38;5;220m\]@\[$(tput sgr0)\]\[\033[38;5;45m\]\h\[$(tput sgr0)\]\[\033[38;5;231m\]:\w\[$(tput sgr0)\] \[$(tput sgr0)\]\[\033[38;5;155m\]\\$\[$(tput sgr0)\] \[$(tput sgr0)\]"
```

## Aliases for fast log tails
```plain
alias t='tail -f /var/log/syslog | ccze'
alias m='tail -f /var/log/mail.log | ccze'
alias me='tail -f /var/log/mail.err | ccze'
alias r='tail -f /var/log/rspamd/rspamd.log | ccze'
alias f='tail -f /var/log/fail2ban.log | ccze'
```

## Colorful ls outputs (optional)
```plain
export LS_OPTIONS='--color=auto'
eval "$(dircolors)"
alias ls='ls $LS_OPTIONS'
alias ll='ls $LS_OPTIONS -l'
alias l='ls $LS_OPTIONS -lA'
```