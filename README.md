Acme-Tiny Automation for Debian
===============================


Overview
--------
[Let's Encrypt](https://letsencrypt.org) provides
SSL certificates for your services.  
Server validation and certificate fetching is done via the acme protocol,
for which a number of clients exists.

[Acme-tiny](https://github.com/diafygi/acme-tiny)
is one of the lightweight tools among them.  
The author keeps the whole script below 200 lines to allow
for easy auditing.  
To achieve this, acme-tiny implements only the core actions of the
acme protocol.

For automated system integration further setup and
supporting scripts are required.  
This repository provides these for Apache running on Debian
along with complete setup instructions.  
They should also work also with other Linux systems, maybe with some
additional tweaking. If you found out what needs to be changed
just let me know (Mail, Issue, Pull-Request).
The same applies of course also for other issues or improvements.


Letsencrypt
-----------
Some reading about Letsencrypt (from 2018-12)
- How it works: https://letsencrypt.org/how-it-works/
- Privacy Policy: https://letsencrypt.org/privacy/ - notable points:
  - Stored information: IP addresses from which you access the
    Let’s Encrypt service; all resolved IP addresses for any
    domain names requested; server information related to any
    validation requests; full logs of all inbound HTTP / ACME
    requests, all outbound validation requests; and information
    sent by or inferred from your client software.
  - We will store this information for a minimum of seven years
  - By providing your email address, you are consenting to receive
    service-related emails from us.
- Subscriber Agreement:
  https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf


Server Preparation
------------------

##### Installation
1. Install acme-tiny, e.g.:  
   `sudo aptitude install acme-tiny`

##### User and directory setup
1. Create user and group acme  
  `adduser --no-create-home --disabled-login --group acme`  
  `adduser --no-create-home --disabled-login --ingroup acme acme`
1. Change login shell in /etc/passwd for acme to /usr/sbin/nologin
1. Create config directory  
   `mkdir /etc/acme-tiny`  
   `chown acme:root /etc/acme-tiny`  
   `chmod u=rwx,og= /etc/acme-tiny`
1. Challenge is placed in webroot /.well-known/acme-challenge  
    `mkdir /srv/support/acme-challenge`  
    `chown acme:www-data /srv/support/acme-challenge`  
    `chmod u=rwx,g=rx,o= /srv/support/acme-challenge`

##### Setup Apache Web Server
1. Serve directory /.well-known/acme-challenge/
   as alias to the challenge dir  
   Create file [acme.conf](apache/acme.conf) as
    /etc/apache2/sites-available/acme.conf
1.  Include this file in all site files, e.g.  
    `# ACME-Challenge directory`  
    `include /etc/apache2/sites-available/acme.conf`


##### Setup acme-tiny
1. `cd /etc/acme-tiny`
1. Create private account key  
    `openssl genrsa 4096 > account.key`
1. Generate domain private key(s) for each domain  
    `openssl genrsa 4096 > DOMAINNAME.key`
1. Create CSR for domain(s) for each domain
    - For a single domain (DOMAINNAME)  
      `openssl req -new -sha256 -key DOMAINNAME.key -subj "/CN=DOMAINNAME" > DOMAINNAME.csr`
    - For multiple domains (DOMAINNAME, sub1.DOMAINNAME, sub2.DOMAINNAME)  
      `openssl req -new -sha256 -key DOMAINNAME.key -subj "/" -reqexts SAN \
        -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:DOMAINNAME,DNS:sub1.DOMAINNAME,DNS:sub2.DOMAINNAME")) > DOMAINNAME.csr`
1. Adapt permissions  
   `chown root:root *.key`  
   `chown acme:acme account.key *.csr`  
   `chmod u=r,og=   *.key *.csr`

##### Test Setup
1. Manually fetch a signed certificate  
    `sudo -u acme sh -c "acme-tiny --account-key ./account.key --csr ./DOMAINNAME.csr --acme-dir /srv/support/acme-challenge/ > ./DOMAINNAME.crt"`
1. Update permissions
    `chmod u=r,og=  /etc/acme-tiny/*.crt`
1. Reload consumers, for example  
    Apache:  `service apache2 graceful`  
    Dovecot: `service dovecot reload`  
    Postfix: `service postfix reload`


Automate Certificate Renewal
----------------------------

##### Service restarter
1. Create file [acme_restart_services](acme_restart_services) as
   /usr/local/sbin/acme_restart_services
1. Update the services to restart and the restart commands
   at the end of the file
1. Adapt permissions  
   `chown root:root /usr/local/sbin/acme_restart_services`  
   `chmod u=rwx,og=rx /usr/local/sbin/acme_restart_services`
1. Allow service restart from acme user: visudo (/etc/sudoers)  
   `# acme user can call service restart script without password`  
   `acme	ALL = (root) NOPASSWD: /usr/local/sbin/acme_restart_services`

##### Logfile
1. Create logfiles with correct permissions  
  `touch /var/log/acme_cron_update.log`  
  `chown root:acme /var/log/acme_cron_update.log`  
  `chmod ug=rw,o= /var/log/acme_cron_update.log`
1. Create logrotate config file
   [acme-tiny](logrotate.d/acme-tiny) as
   /etc/logrotate.d/acme-tiny
1. Adapt permissions  
   `chown root:root /etc/logrotate.d/acme-tiny`  
   `chmod u=rw,og=r /etc/logrotate.d/acme-tiny`

##### Automation script
1. Create file [acme_cron_update](acme_cron_update) as
   /usr/local/sbin/acme_cron_update
1. Adapt user settings  
   At the top of the file, below "User Settings"
1. Adapt permissions  
   `chown root:root /usr/local/sbin/acme_cron_update`  
   `chmod u=rwx,og=rx /usr/local/sbin/acme_cron_update`
1. Test with manual execution  
   `sudo -u acme sh -c '/usr/local/sbin/acme_cron_update 2>&1 | tee -a /var/log/acme_cron_update.log'`

##### Cron entry
1. Create file [acme-tiny](cron.d/acme-tiny) as
   /etc/cron.d/acme-tiny
1. For testing purposes enable the weekly update line instead
   of the monthly update
1. Adapt permissions  
   `chown root:root /etc/cron.d/acme-tiny`  
   `chmod u=rw,og=r /etc/cron.d/acme-tiny`
1. Cron sends the job output automatically by mail to root,
   ensure that correct mail forwarding to your email account is in place


License & Author
----------------
Acme-Tiny Automation  
Copyright (C) 2019 Robert Lange (https://sethdepot.org/site/RoLa.html)

This program is free software; you can redistribute it and/or  
modify it under the terms of the GNU General Public License  
version 3 as published by the Free Software Foundation.

This program is distributed in the hope that it will be useful,  
but WITHOUT ANY WARRANTY; without even the implied warranty of  
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the  
GNU General Public License for more details.

License Text: [LICENSE-gpl-3.0.md](LICENSE-gpl-3.0.md)


Web Links
---------
- Repository  
  https://github.com/sd2k9/acme-tiny-automation
- Issue tracker  
  https://github.com/sd2k9/acme-tiny-automation/issues
