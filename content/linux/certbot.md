---
title: "Certbot"
date: 2023-10-03T10:40:17Z
draft: False
---

Certbot lets you create free SSL certficate from Let's Encrypt.

{{< toc >}}

## Install certbot and the Cloudflare Plugin
```
apt install certbot python3-certbot-dns-cloudflare
```

## Setup certbot and request a wildcard certificate
Create a API token at CloudFlare and setup a certbot config file e.g. `certbot-creds.ini`.
```
# Cloudflare API credential used by Certbot
dns_cloudflare_api_token = API_TOKEN
```

Secure the file, so only root can read it.
```
chown root:root certbot-creds.ini
chmod 600 certbot-creds.ini
```

## Request your first wildcard certificate
```plain
certbot certonly --dns-cloudflare --dns-cloudflare-credentials /path/to/certbot-creds.ini -d '*.your-domain.tld'
```

Your newly created certificate is located under `/etc/letsencrypt/live/`
The renewal process is managed by a systemd timer, which was installed by the certbot package.