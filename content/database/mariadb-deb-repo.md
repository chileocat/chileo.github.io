---
title: "MariaDB Debian Repository"
date: 2023-10-03T10:40:17Z
draft: False
---

Install MariaDB from the offical repository.

### MariaDB Repository
```
sudo apt install apt-transport-https curl
sudo curl -o /etc/apt/keyrings/mariadb-keyring.pgp 'https://mariadb.org/mariadb_release_signing_key.pgp'
```

Create a new file under `/etc/apt/sources.list.d/`, e.g. `mariadb.sources`. If you need another mirror, check out https://mariadb.org/download/?t=repo-config.
```
# MariaDB 11.1 repository list - created 2023-09-12 12:07 UTC
# https://mariadb.org/download/
X-Repolib-Name: MariaDB
Types: deb
# deb.mariadb.org is a dynamic mirror if your preferred mirror goes offline.
# See https://mariadb.org/mirrorbits/ for details.
# URIs: https://deb.mariadb.org/11.1/debian
URIs: https://mirror.mva-n.net/mariadb/repo/11.1/debian
Suites: bookworm
Components: main
Signed-By: /etc/apt/keyrings/mariadb-keyring.pgp
```

Now install the server and client.
```
apt update
apt install mariadb-server mariadb-client
```