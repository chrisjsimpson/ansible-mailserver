# Ansible-mailserver

Ansible configured Postfix & dovecot server

A fullstack but simple mail server (smtp, imap, antispam). Only configuration files, no SQL database. Keep it simple and versioned. Easy to deploy and upgrade. (Inspired by cd)


## Setup server
On an ubuntu lts server:

```
apt update
apt install -y git ansible
git clone https://github.com/chrisjsimpson/ansible-mailserver.git
cd ansible-mailserver/

```
## How to run

Configure ansible inventory file with your host (e.g. ip of a vps instance https://www.vultr.com/?ref=8472727)

If you're running ansible directly on the server:
```
[mailservers]
localhost ansible_user=root ansible_connection=local
```

Otherwise:

```
[mailservers]
<your mailserver ip> ansible_user=root
```

## Configure DNS (so that cert generation passes)

- A record for `email.example.co.uk`
- MX record pointing to A record

## Run Playboks

- Certs (uses certbot to get a certificate) 
- Mail server configuration


Run the cert playbook to generate a cert for the server:
```
ansible-playbook -i playbooks/inventory.yaml -e account_email=postmaster@example.com -e domain=email.example.com playbooks/certs/certs.yaml
```

Where

- `-i` is the inventory.yaml file
- `-e account_email=postmaster@example.com` the email address used for certbot terms of service
- `-e domain=email.example.com` the domain of the email server you want a certificate for
- `playbooks/certs.yaml` contains the automated steps to get a certificate (aka a playbook)

Success looks like:
```
PLAY RECAP ************************************************************************************************************************
localhost                  : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```


Run the mail server playbook:

This will configure your server with postfix, dovecot, opendkim and
spamassassin.

```
ansible-playbook -i playbooks/inventory.yaml -e myhostname=email.example.com -e postmaster_address=postmaster@example.com -e generate_dkim_keys=True playbooks/mailserver/mail-server.yaml
```

DKIM will fail:
```
TASK [Make opendkim the owner of dkim all keys] ***********************************************************************************
failed: [localhost] (item=example.co.uk) => {"ansible_loop_var": "item", "changed": false, "item": "example.co.uk", "msg": "file (/etc/opendkim/keys/example.co.uk/default.private) is absent, cannot continue", "path": "/etc/opendkim/keys/example.co.uk/default.private"}

PLAY RECAP ************************************************************************************************************************
localhost                  : ok=23   changed=22   unreachable=0    failed=1    skipped=1    rescued=0    ignored=0 
```

## (re)Generate DKIM keys for a single domain

Update `playbooks/group_vars/mailservers.yaml`, setting `virtual_mailbox_domains` and
`regenerate_dkim_keys_domains`. Then ansible will only generate new
keys for the domain in `regenerate_dkim_keys_domains`. 

e.g:
```
virtual_mailbox_domains: ['example.com']
regenerate_dkim_keys_domains: ['example.com']
```

Then run the playboook:

```
ansible-playbook -i playbooks/inventory.yaml playbooks/dkim/dkim-keys-generation.yaml
```

Run the cert renewal for crontab entry

```
ansible-playbook -i playbooks/inventory.yaml -e account_email=postmaster@example.com -e domain=email.example.com playbooks/certs/cron-cert.yaml
```

Re-run mail-server playbook
```
ansible-playbook -i playbooks/inventory.yaml -e myhostname=email.example.com -e postmaster_address=postmaster@example.com -e generate_dkim_keys=True playbooks/mailserver/mail-server.yaml
```

Success looks like:
```
PLAY RECAP ************************************************************************************************************************
localhost                  : ok=39   changed=19   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0  
```

## Configure DNS DKIM and SPF

Get DKIM keys from `/etc/opendkim/keys/` on the server.

- Add DKIM record 
- Add SPF record e.g. `v=spf1 ip4:192.168.1.1 mx ~all`
- Add DMARC record e.g. `v=DMARC1; p=none; pct=100; fo=1; rua=mailto:dmarck-reports@example.co.uk` (use a virtual)


# Add user(s)

Note this script is not idempotent.
```
#!/bin/bash

set -exo pipefail

if [ "$#" -ne 3 ];then
  echo "Usage: $0 DOMAIN USERNAME PASSWORD"
fi

DOMAIN=$1
USERNAME=$2
PASSWORD=$3

MAIL_USER_ID=`id -u mail`

echo Creating mail folder for $USERNAME@$DOMAIN
mkdir -p /var/mail/vhosts/$DOMAIN/$USERNAME

echo Setting mail folder permissions
chown -R mail:mail /var/mail/vhosts/$DOMAIN/$USERNAME

echo Generating password hash
PASSWORD_HASH=`doveadm pw -s SHA512-CRYPT -p $PASSWORD`

echo Adding $USERNAME@$DOMAIN to /etc/dovecot/users file

echo $USERNAME@$DOMAIN:$PASSWORD_HASH:$MAIL_USER_ID:$MAIL_USER_ID::/var/mail/vhosts/$DOMAIN >> /etc/dovecot/users

echo Add vmailbox to postfix
echo $USERNAME@DOMAIN $DOMAIN/$USERNAME >> /etc/postfix/vmailbox

echo Run postmap
postmap /etc/postfix/vmailbox

echo Reload postfix
systemctl reload postfix
echo Reloading dovecot
systemctl reload dovecot.service

```


### Useful resources:

- ipv6 / ipv4 gmail sending https://blog.hqcodeshop.fi/archives/122-Fixing-Googles-new-IPv6-mail-policy-with-Postfix.html 

- https://www.linuxbabe.com/mail-server/setting-up-dkim-and-spf
- https://www.linuxbabe.com/mail-server/create-dmarc-record
- https://www.linuxbabe.com/mail-server/setup-basic-postfix-mail-sever-ubuntu 
- https://www.linuxbabe.com/mail-server/block-email-spam-check-header-body-with-postfix-spamassassin
