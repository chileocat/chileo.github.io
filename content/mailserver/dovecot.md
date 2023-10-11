---
title: "Setting up Dovecot"
date: 2023-10-11T13:46:25Z
draft: False
---

We have to configure Dovecot for receiving mails from Postfix, also getting it's information from the database.

## Create Mailuser
```plain
groupadd -g 9000 vmail
useradd -g vmail -u 9000 vmail -d /var/vmail -m
chown -R vmail:vmail /var/vmail

```

## Dovecot settings
### 10-auth.conf
If you need Outlook, add also `login` as mechanism.

```plain
auth_mechanisms = plain
```

Also comment the auth-system.conf.ext and uncomment the auth-sql.conf.ext.
```plain
#!include auth-system.conf.ext
!include auth-sql.conf.ext
#!include auth-ldap.conf.ext
#!include auth-passwdfile.conf.ext
#!include auth-checkpassword.conf.ext
#!include auth-static.conf.ext
```

### dovecot-sql.conf.ext
```plain
password_query = \
   SELECT password FROM accounts WHERE username = '%Ln' AND domain = '%Ld' AND enabled = TRUE

user_query = \
   SELECT CONCAT(username,'@',domain) AS user,\
   CONCAT('*:storage=', quota, 'M') AS quota_rule, \
   '/var/vmail/%Ld/%Ln' AS home, \
   9000 AS uid, 9000 AS gid\
   FROM accounts WHERE username = '%Ln' AND domain = '%Ld' AND sendonly = FALSE

iterate_query = SELECT CONCAT(username, '@', domain) AS user FROM accounts WHERE sendonly = FALSE
```

Set right permissions on this file.
```plain
chown root:root /etc/dovecot/dovecot-sql.conf.ext
chown 600 /etc/dovecot/dovecot-sql.conf.ext```

### 10-mail.conf
Change the mail_location to
```plain
mail_location = maildir:~/Maildir:LAYOUT=fs
```

Also enable the quota plugin.
```plain
mail_plugins = quota
```

### 10-master.conf
Enable Postfix smtp-auth
```plain
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
```

Enable the LMTP daemon.
```plain
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }

  # Create inet listener only if you can't use the above UNIX socket
  #inet_listener lmtp {
    # Avoid making LMTP visible for the entire internet
    #address =
    #port =
  #}
}
```

### 10-ssl.conf
```plain
ssl = required

ssl_cert = </etc/letsencrypt/live/chileo.net/cert.pem
ssl_key = </etc/letsencrypt/live/chileo.net/privkey.pem

ssl_dh = </etc/dovecot/dh4096.pem
ssl_min_protocol = TLSv1.2
ssl_cipher_list = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305
ssl_prefer_server_ciphers = no
```

## Enable server-side rules for LMTP (20-lmtp.conf)
```plain
protocol lmtp {
  # Space separated list of plugins to load (default is global mail_plugins).
  mail_plugins = $mail_plugins sieve
}
```

### 15-mailboxes.conf
```plain
namespace inbox {
  # These mailboxes are widely used and could perhaps be created automatically:
  mailbox Drafts {
    special_use = \Drafts
  }
  mailbox Junk {
    special_use = \Junk
    auto = subscribe
    #autoexpunge = 30d
  }
  mailbox Trash {
    special_use = \Trash
    auto = subscribe
    #autoexpunge = 30d
  }

  # For \Sent mailboxes there are two widely used names. We'll mark both of
  # them as \Sent. User typically deletes one of them if duplicates are created.
  mailbox Sent {
    special_use = \Sent
    auto = subscribe
  }
  mailbox "Sent Messages" {
    special_use = \Sent
  }
```

### 20-imap.conf
```plain
mail_plugins = $mail_plugins imap_quota imap_sieve
mail_max_userip_connections = 25
```

### 20-lmtp.conf
```plain
mail_plugins = $mail_plugins sieve
```

### 20-managesieve.conf
Set managesive to listen only on localhost.
```plain
service managesieve-login {
  inet_listener sieve {
    address = localhost
    port = 4190
  }
```

### 90-quota.conf
```plain
plugin {
  quota_warning = storage=95%% quota-warning 95 %u
  quota_warning2 = storage=80%% quota-warning 80 %u
}

service quota-warning {
   executable = script /usr/local/bin/quota-warning.sh
   unix_listener quota-warning {
     user = vmail
     group = vmail
     mode = 0660
   }
}
```

```plain
plugin {
  #quota = dirsize:User quota
  quota = maildir:User quota
  #quota = dict:User quota::proxy::quota
  #quota = fs:User quota

  quota_status_success = DUNNO
  quota_status_nouser = DUNNO
  quota_status_overquota = "452 4.2.2 The email account that you tried to reach is over quota and cannot receive any more emails"
}
```

```plain
service quota-status {
  executable = /usr/lib/dovecot/quota-status -p postfix
  unix_listener /var/spool/postfix/private/quota-status {
    user = postfix
  }
}
```

### quota-warning.sh
Create a new file under `/usr/local/bin` named `quota-warning.sh`

```bash
#!/bin/sh
PERCENT=$1
USER=$2
cat << EOF | /usr/lib/dovecot/dovecot-lda -d $USER -o "plugin/quota=maildir:User quota:noenforcing"
From: postmaster@chileo.ch
Subject: Quota warning - $PERCENT% reached

Your mailbox can only store a limited amount of emails.
Currently it is $PERCENT% full. If you reach 100% then
new emails cannot be stored. Thanks for your understanding.
EOF
```

```plain
chmod +x /usr/local/bin/quota-warning.sh
```

### 90-quota.conf
```plain
sieve_after = /etc/dovecot/sieve-after

# From elsewhere to Junk folder
imapsieve_mailbox1_name = Junk
imapsieve_mailbox1_causes = COPY
imapsieve_mailbox1_before = file:/etc/dovecot/sieve/learn-spam.sieve

# From Junk folder to elsewhere
imapsieve_mailbox2_name = *
imapsieve_mailbox2_from = Junk
imapsieve_mailbox2_causes = COPY
imapsieve_mailbox2_before = file:/etc/dovecot/sieve/learn-ham.sieve

sieve_pipe_bin_dir = /etc/dovecot/sieve
sieve_global_extensions = +vnd.dovecot.pipe
sieve_plugins = sieve_imapsieve sieve_extprograms
```



### spam-to-folder.sieve

```plain
mkdir /etc/dovecot/sieve-after
mkdir /etc/dovecot/sieve
```

```plain
require ["fileinto"];

if header :contains "X-Spam" "Yes" {
 fileinto "Junk";
 stop;
}
```

```plain
sievec /etc/dovecot/sieve-after/spam-to-folder.sieve
```

### learn-spam.sieve
```plain
require ["vnd.dovecot.pipe", "copy", "imapsieve"];
pipe :copy "rspamd-learn-spam.sh";
```

### learn-ham.sieve
```
require ["vnd.dovecot.pipe", "copy", "imapsieve", "variables"];
if string "${mailbox}" "Trash" {
  stop;
}
pipe :copy "rspamd-learn-ham.sh";
```

```plain
sievec /etc/dovecot/sieve/learn-spam.sieve
sievec /etc/dovecot/sieve/learn-ham.sieve
chmod 600 /etc/dovecot/sieve/learn-{spam,ham}.{sieve,svbin}
chown vmail:vmail /etc/dovecot/sieve/learn-{spam,ham}.{sieve,svbin}
```

### rspamd-learn-spam.sh
```bash
#!/bin/sh
exec /usr/bin/rspamc learn_spam
```

### rspamd-learn-ham.sh
```bash
#!/bin/sh
exec /usr/bin/rspamc learn_ham
```

```plain
chmod 700 /etc/dovecot/sieve/rspamd-learn-{spam,ham}.sh
chown vmail:vmail /etc/dovecot/sieve/rspamd-learn-{spam,ham}.sh
```