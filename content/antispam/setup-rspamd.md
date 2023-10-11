---
title: "Setting up Dovecot"
date: 2023-10-11T13:46:25Z
draft: False
---

At least we need some reliable rspamd settings for a good spam-filtering.

## local.d/actions.conf
```plain
reject = 9;
add_header = 4;
greylist = 3;
```

## local.d/arc.conf
```plain
allow_username_mismatch = true;
#sign_networks = "/etc/rspamd/sign_networks.map";
path = "/var/lib/rspamd/dkim/$domain.$selector.key";
selector_map = "/etc/rspamd/maps.d/dkim_selectors.map";
```

## local.d/blacklist_from.map
Leave it emtpy for now.

## local.d/blacklist_ip.map
Leave it emtpy for now.

## local.d/classifier-bayes.conf
```plain
servers = "127.0.0.1";
backend = "redis";
expire = 315360000;
```

## local.d/dkim-signing.conf
```plain
allow_username_mismatch = true;
#sign_networks = "/etc/rspamd/sign_networks.map";
path = "/var/lib/rspamd/dkim/$domain.$selector.key";
selector_map = "/etc/rspamd/maps.d/dkim_selectors.map";
```

## local.d/greylist.conf
```plain
servers = "127.0.0.1";
```

## local.d/logging.inc
```plain
debug_modules = ["dkim_signing"];
```

## local.d/whitelist_from.map
```plain
vincent.chileo.ch
vincent.chileo.net
jarvis.chileo.net
```

## local.d/whitelist_ip.map
```plain
87.102.193.61
```

## local.d/worker-controller.inc
password = "secret";

## local.d/worker-proxy.inc
```plain
milter = yes;
upstream "local" {
  self_scan = yes; # Enable self-scan
}
```

## override.d/classifier-bayes.conf
```plain
autolearn = true;
```

## override.d/milter_headers.conf
```plain
extended_spam_headers = true;
use = ["x-spam-level", "x-spam-status", "authentication-results"];
```

## override.d/redis.conf
```plain
servers = "127.0.0.1";
```

## Create new DKIM keys
```apt update
apt install apt install python3-dkim
mkdir /var/lib/rspamd/dkim
cd /var/lib/rspamd/dkim
dknewkey $domain.$selector
rm $domain.$selector.dns
chown _rspamd:_rspamd *
```

## maps.d/dkim_selectors.map
```plain
chileo.ch 20231001
chileo.net 20231001
```