# Setup Dovecot and Postfix for Thunderbird development

When developing Thunderbird, we often need to send lots of test mails. It's helpful to setup Dovecot (IMAP server) and Postfix (SMTP server) on local. This is a note on setting them up on Fedora.

## Setup Dovecot

**Install** Dovecot

```
sudo dnf install dovecot
```

**Edit** `/etc/dovecot/conf.d/10-auth.conf`

Uncomment the `!include auth-passwdfile.conf.ext` line.
Comment other `#!include auth-*.conf.ext` lines.

**Edit** `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`, put in

```
passdb {
  driver = passwd-file
  args = scheme=plain username_format=%n /etc/dovecot/users
}

userdb {
  driver = passwd-file
  args = username_format=%n /etc/dovecot/users
}
```

**Edit** `/etc/dovecot/users`, put in

```
rnons:rnons:1000:1000
test:test:1000:1000
```

It means creating two IMAP users: `rnons` and `test`. `rnons` is the current Fedora user, `test` will be created later when setting up Postfix.

See https://doc.dovecot.org/configuration_manual/authentication/user_databases_userdb about the meaning of each field.

**Edit** `/etc/dovecot/conf.d/10-mail.conf`, set

```
mail_location = mbox:/home/rnons/dovecot/%n:INBOX=/var/mail/%n
```

It means all IMAP mail folders are in `~/dovecot/$IMAP_USER_NAME`. And the `INBOX` folder is the system mail folder.

## Setup Postfix

**Install** Postfix

```
sudo dnf install postfix
```

**Edit** `/etc/dovecot/conf.d/20-submission.conf`, set

```
submission_relay_host = localhost
```

**Create** test user

```
sudo useradd -s /sbin/nologin -M test
sudo chmod o+r /var/mail/test
```

## Add the two email accounts to Thunderbird

**Restart** Dovecot and Postfix

```
sudo systemctl restart dovecot
sudo systemctl restart postfix
```

**Add** email accounts to Thunderbird

Config \ Account | rnons | test
--|--|--
Name | rnons | test
Email | rnons@localhost | test@localhost
Password | rnons | test

**Try** sending a mail from rnons@localhost to test@localhost and vice versa. If not working, try editing server configs manually

Config \ Server | Incoming (IMAP) | Outgoing (SMTP)
--|--|--
Server | localhost | localhost
Port | 143 | 587
SSL | None | None
Authentication | Normal password | Normal password
Username | rnons | test

## Extra: setup SASL to test CRAM-MD5

**Edit** `/etc/dovecot/conf.d/10-master.conf`, set

```
unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = postfix
}
```

**Edit** `/etc/dovecot/conf.d/10-auth.conf`, set

```
auth_mechanisms = plain login cram-md5
```

**Edit** `/etc/postfix/main.cf`, put in

```
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
```

**Restart** dovecot/postfix and in Thunderbid, change the server authentication to `CRAM-MD5`.

See http://www.postfix.org/SASL_README.html for more details.

## Extra: setup SASL to test GSSAPI / Kerberos

Setup SASL the same as the previous CRAM-MD5 section, then

**Edit** `/etc/dovecot/conf.d/10-auth.conf`, set

```
auth_mechanisms = plain login cram-md5 gssapi
```

### Setup Kerberos

**Install** Kerberos

```
sudo dnf install krb5-server krb5-workstation
```

**Edit** `/etc/krb5.conf`, put in

```
[logging]
    default = FILE:/var/log/krb5kdc.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/krb5kdc.log

[libdefaults]
    default_realm = TB.DEV

[realms]
TB.DEV = {
    kdc = localhost
    admin_server = localhost
}

[domain_realm]
.localhost = TB.DEV
localhost = TB.DEV
```

**Create** Kerberos DB

```
sudo kdb5_util -r TB.DEV create -s
```

**Start** Kerberos

```
sudo krb5kdc
```

**Config** Kerberos

```sh
sudo kadmin.local

#-- inside the kadmin repl --#
# add a principle for the current Fedora user
addprinc rnons
# add a principle for the SMTP service
addprinc -randkey smtp/localhost
ktadd smtp/localhost
# add a principle for the IMAP service
addprinc -randkey imap/localhost
ktadd imap/localhost
```

**Login** Kerberos

```
kinit
klist
```

See https://wiki.archlinux.org/index.php/Kerberos for more details.

### Setup Dovecot

**Copy** keytab

```
sudo cp /etc/krb5.keytab /etc/dovecot
sudo chgrp dovecot /etc/dovecot/krb5.keytab
sudo chmod g+r /etc/dovecot/krb5.keytab
```

**Check** keytab (optional)

```
klist -Kek /etc/dovecot/krb5.keytab
```

**Edit** `/etc/dovecot/conf.d/10-auth.conf`, set

```
auth_gssapi_hostname = "$ALL"
auth_mechanisms = gssapi
auth_krb5_keytab = /etc/dovecot/krb5.keytab
```

**Restart** dovecot/postfix and in Thunderbid, change the server authentication to `Kerberos / GSSAPI`.

The dovecot wiki https://wiki.dovecot.org/Authentication/Kerberos mentions Samba, but it's not really needed.
