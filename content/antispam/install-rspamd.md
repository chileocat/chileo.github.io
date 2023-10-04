---
title: "Install Rspamd"
date: 2023-10-03T10:40:17Z
draft: False
---

Install Rspamd from the offical repository.

### Rspamd Repository
```plain
sudo apt install lsb-release wget gpg
CODENAME=`lsb_release -c -s`
sudo mkdir -p /etc/apt/keyrings
wget -O- https://rspamd.com/apt-stable/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/rspamd.gpg > /dev/null
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main" |sudo tee /etc/apt/sources.list.d/rspamd.list
echo "deb-src [arch=amd64 signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main" | sudo tee -a /etc/apt/sources.list.d/rspamd.list
```

## Update and install Rspamd
```
apt update
apt install rspamd
```