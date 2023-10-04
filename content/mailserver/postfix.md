---
title: "Setting up Postfix"
date: 2023-10-03T10:40:25Z
draft: False
---

Postfix has to be configured for getting it's information from the database. Also we need to set some security settings, for preventing getting used as open-relay or receiving a lot of spam.

## Postfix configuration (main.cf)
{{< expand "main.cf" >}}
```plain
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 2 on
# fresh installs.
compatibility_level = 3.6

# TLS parameters
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes
smtpd_tls_cert_file = /etc/letsencrypt/live/chileo.net/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/chileo.net/privkey.pem
smtpd_tls_mandatory_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_protocols = !SSLv2, !SSLv3, !TLSv1, !TLSv1.1
smtpd_tls_mandatory_ciphers = medium
smtpd_tls_dh1024_param_file = /etc/postfix/dhparam.pem
tls_medium_cipherlist = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
tls_preempt_cipherlist = no

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

myhostname = vincent.chileo.net
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, vincent.chileo.net, localhost
mynetworks = 127.0.0.0/8 [::1]/128 IPV4/32 [IPv6]/128 172.23.0.0/24
mailbox_size_limit = 0
message_size_limit = 52428800
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_mailbox_domains = mysql:/etc/postfix/sql.d/domains.cf
virtual_mailbox_maps = mysql:/etc/postfix/sql.d/accounts.cf
virtual_alias_maps = mysql:/etc/postfix/sql.d/aliases.cf,mysql:/etc/postfix/sql.d/email2email.cf

##
## rspamd settings
##
smtpd_milters = inet:127.0.0.1:11332
non_smtpd_milters = inet:127.0.0.1:11332
milter_mail_macros = i {mail_addr} {client_addr} {client_name} {auth_authen}

##
## Mail-Queue Einstellungen
##

maximal_queue_lifetime = 1h
bounce_queue_lifetime = 1h
maximal_backoff_time = 15m
minimal_backoff_time = 5m
queue_run_delay = 5m

### Bedingungen, damit Postfix ankommende E-Mails als Empfängerserver entgegennimmt (zusätzlich zu relay-Bedingungen)
### check_recipient_access prüft, ob ein account sendonly ist
smtpd_recipient_restrictions = permit_sasl_authenticated
                               permit_mynetworks
                               reject_unknown_recipient_domain
                               check_recipient_access proxy:mysql:/etc/postfix/sql.d/recipient-access.cf
                               check_policy_service unix:private/quota-status
                               reject_unauth_destination

### Bedingungen, damit Postfix als Relay arbeitet (für Clients)
smtpd_relay_restrictions = permit_sasl_authenticated
                           permit_mynetworks
                           reject_non_fqdn_recipient
                           reject_unknown_recipient_domain
                           defer_unauth_destination

### Bedingungen, die SMTP-Clients erfüllen müssen (sendende Server)
smtpd_client_restrictions = permit_sasl_authenticated
                            permit_mynetworks
                            check_client_access hash:/etc/postfix/without_ptr
                            reject_unknown_client_hostname

### Wenn fremde Server eine Verbindung herstellen, müssen sie einen gültigen Hostnamen im HELO haben.
smtpd_helo_required = yes
smtpd_helo_restrictions = permit_sasl_authenticated
                          permit_mynetworks
                          reject_non_fqdn_helo_hostname
                          reject_invalid_helo_hostname
```
{{< /expand >}}

## Making Postfix get infos from the database
Create the following files under `/etc/postfix/sql.d`.

### accounts.cf
```plain
user = mailserver_ro
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT 1 as FOUND FROM accounts WHERE username = '%u' AND domain = '%d' AND enabled = TRUE LIMIT 1;
```

### aliases.cf
```plain
user = mailserver_ro
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT destination FROM aliases WHERE CONCAT(source_username, '@', source_domain) = '%s';
```

### domains.cf
```plain
user = mailserver_ro
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT 1 as FOUND FROM domains WHERE name='%s'
```

### email2email.cf
```plain
user = mailserver_ro
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT * FROM (SELECT CONCAT(username, '@', domain) AS email FROM accounts) BASE WHERE email = '%s';
```

### recipient-access.cf
```plain
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT IF(sendonly = TRUE, 'REJECT', 'OK') AS access FROM accounts WHERE username = '%u' AND domain = '%d' AND enabled = TRUE LIMIT 1;
```

### sender-login-maps.cf
```plain
user = mailserver_ro
password = ...
hosts = 127.0.0.1
dbname = mailserver
query = SELECT CONCAT(username, '@', domain) AS 'owns' FROM accounts WHERE username = '%u' AND domain = '%d' and enabled = true UNION SELECT CONCAT(source_username,'@',source_domain) AS 'owns' FROM aliases WHERE destination = '%s' AND enabled = TRUE;
```

## Enable submission in Postfix
For this, edit the `master.cf` file and add/uncomment the following
```plain
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
```

## Whitelist file for some systems without any PTR record
Sometime you want to whitelist some IP addresses without any valid PTR, create a new emtpy file unter `/etc/postfix` named `without_ptr` and add IPs in the following scheme
```plain
127.0.0.1  OK
```

Hash the file with `postmap /etc/postfix/without_ptr`, so will get a `without_ptr.db` file which is usable by Postfix.

## Check the configuration
Check if postfix doesn't reports any error with `postfix check` after then reload Postfix with the new settings.
```plain
systemctl reload postfix
```

You can check, if Postfix is able to read from the database, e.g. mailbox accounts.
```plain
postmap -q some@email.com mysql:/etc/postfix/sql.d/accounts.cf
```

This should report 1, if the mailbox is found in the database, otherwis nothing.