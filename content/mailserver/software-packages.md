---
title: "Install the Software packages"
date: 2023-10-03T10:40:25Z
draft: False
---

Let us install the necessary Debian packages to make it an actual mail server.

```plain
sudo apt update
sudo apt install postfix postfix-mysql redis-server swaks dovecot-mysql dovecot-imapd \
                 dovecot-managesieved dovecot-lmtpd ca-certificates
```

For the moment, keep the default answers on postfix. We will reconfigure it later.