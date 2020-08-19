# Own full featured mailserver with unlimited users, aliases and domains

## Prerequesites
- CentOS 7
- A domain

Start by installing repos and packages

```
# yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
# yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm -y
# curl https://rspamd.com/rpm-stable/centos-7/rspamd.repo > /etc/yum.repos.d/rspamd.repo
# rpm --import https://rspamd.com/rpm-stable/gpg.key
```
Disable SELinux
```
# yum update -y
# vi /etc/sysconfig/selinux - disable SELinux
# reboot
```
Activate PHP 7.3 repo by setting enable to ```true```
``` 
# vi /etc/yum.repos.d/remi-php73.repo 
```

Install packages
```
# yum install vim htop postfix dovecot httpd certbot python2-certbot-apache mariadb mariadb-server dovecot-mysql wget php phpMyAdmin rspamd dovecot-pigeonhole pypolicyd-spf opendkim perl-Mail-DKIM opendmarc nmap swaks opendbx-mysql unzip
```
Enable MariaDB and set root password
```
# systemctl start mariadb
# systemctl enable mariadb
# mysql_secure_installation 
```
Create MySQL config file in your user's hoome dir for faster access
```
# vim ~/.my.cnf

[client]
host=localhost
user=root
password=""
```
Create and import database
```
mysqladmin create mailserver
mysql mailserver < mailserver.sql
```
Start and enable Apache
```
systemctl start httpd
systemctl enable httpd
```
Stop the postfix for now
```
systemctl stop postfix
```

Create MySQL users
- mailadmin
- mailuser
- roundcube

Copy all configs to their respective paths

Download and unpack **Roundcube**
```
# wget https://github.com/roundcube/roundcubemail/releases/download/1.4-rc1/roundcubemail-1.4-rc1-complete.tar.gz
# tar -xvzf roundcubemail-1.4-rc1-complete.tar.gz
# mv roundcubemail-1.4-rc1 /var/www/roundcube
# systemctl reload httpd
# chmod 777 -R /var/www/roundcube/temp/
# chmod 777 -R /var/www/roundcube/logs
```

Put DB connection settings in following files:
- /etc/postfix/mysql*
- /etc/dovecot/dovecot-sql.conf.ext
- /etc/opendkim.conf
- /var/www/roundcube/config/config.inc.php
- /var/www/roundcube/password/config.inc.php

Change hostname in following files:
- /etc/opendmarc.conf
- /var/www/roundcube/config/config.inc.php

Reload httpd and get certificate from **letsencrypt**
```
# systemctl reload httpd
# certbot
# mod 755 -R /etc/letsencrypt/
```

Put SSL cert path in following files:
- postfix/main.cf 
- dovecot/conf.d/ssl.conf and 

`chmod +x /usr/local/bin/certbot-post-hook`

Install crontab to renew TLS cert automatically
```
# crontab -e

0 0 * * *       certbot -q --post-hook /usr/local/bin/certbot-post-hook renew
```

Run following commands:
```
# chgrp postfix /etc/postfix/mysql-*.cf
# chmod u=rw,g=r,o= /etc/postfix/mysql-*.cf

# groupadd -g 5000 vmail
# useradd -g vmail -u 5000 vmail -d /var/vmail -m

# mkdir /var/vmail
# chown -R vmail.vmail /var/vmail
# chown root:root /etc/dovecot/dovecot-sql.conf.ext
# chmod go= /etc/dovecot/dovecot-sql.conf.ext

# systemctl start rspamd
# systemctl enable rspamd

# systemctl start opendkim
# systemctl enable opendkim

# systemctl start opendmarc
# systemctl enable opendmarc

# systemctl start dovecot
# systemctl enable dovecot

# systemctl restart postfix
# systemctl enable postfix

# sievec /etc/dovecot/sieve-after/spam-to-folder.sieve
```
Install spam filtering database:
```
# systemctl stop rspamd
# wget https://rspamd.com/rspamd_statistics/bayes.ham.sqlite
# wget https://rspamd.com/rspamd_statistics/bayes.spam.sqlite
# mv -f *.sqlite /var/lib/rspamd/
# chown _rspamd:_rspamd /var/lib/rspamd/*.sqlite
# systemctl start rspamd
# rspamc stat

# sievec /etc/dovecot/sieve/learn-spam.sieve
# sievec /etc/dovecot/sieve/learn-ham.sieve
# chmod u=rw,go= /etc/dovecot/sieve/learn-{spam,ham}.sieve
# chown vmail.vmail /etc/dovecot/sieve/learn-{spam,ham}.sieve
# chmod u=rwx,go= /etc/dovecot/sieve/rspamd-learn-{spam,ham}.sh
# chown vmail.vmail /etc/dovecot/sieve/rspamd-learn-{spam,ham}.sh

# systemctl restart dovecot
```
Create home folder for GPG:
```
mkdir /var/gnupg
chown apache:apache /var/gnupg/
systemctl reload httpd
```
Run roundcube install to initialize database and then
```
rm -rf /var/www/roundcube/installer/
Change enable_installer to false in roundcube config
Copy persistent_login plugin and run SQL file in sql folder
```

---
To black hole emails to a specific address add:
```
blackhole:      /dev/null
```
to `/etc/aliases`

and then run

`# newaliases`

After that create new alias in DB pointing to just **blackhole**


## How to insert new user in database

```
INSERT INTO users (domain_id, email, password) VALUES ( (SELECT id FROM domains WHERE name='example.com'), 'user@example.com',CONCAT('{SHA256-CRYPT}', ENCRYPT ('your-password', CONCAT('$5$', SUBSTRING(SHA(RAND()), -16)))));
```