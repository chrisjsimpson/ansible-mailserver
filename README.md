# Ansible-mailserver

Ansible configured Postfix & dovecot server

A fullstack but simple mail server (smtp, imap, antispam). Only configuration files, no SQL database. Keep it simple and versioned. Easy to deploy and upgrade. (Inspired by https://github.com/tomav/docker-mailserver)

## How to run

Configure ansible investory file with your host (e.g. ip of a vps instance https://www.vultr.com/?ref=8472727)

```
[mailservers]
192.168.1.1 ansible_user=root
```

## Configure DNS (so that cert generation passes)

- A record for `email.example.co.uk`
- MX record pointing to A record

## Run Playboks

- Certs (uses certbot to get a certificate) 
- Mail server configuration


Run the cert playbook to generate a cert for the server:
```
ansible-playbook playbooks/certs.yaml
```

Run the mail server playbook:

This will configure your server with postfix, dovecot, opendkim and
spamassassin.

```
ansible-playbook playbooks/mail-server.yaml 
```

## Regenerate DKIM keys for a single domain

Update group_vars/mailservers.yaml `virtual_mailbox_domains` and
`regenerate_dkim_keys_domains`. Then ansible will only generate new
keys for the domain in `regenerate_dkim_keys_domains`. 

Then run the playboook:

```
ansible-playbook -i inventory.yaml playbooks/dkim/dkim-keys-generation.yaml
```

## Configure DNS DKIM and SPF

Get DKIM keys from `/etc/opendkim/keys/` on the server.

- Add DKIM record 
- Add SPF record e.g. `v=spf1 ip4:192.168.1.1 mx ~all`
- Add DMARC record e.g. `v=DMARC1; p=none; pct=100; fo=1; rua=mailto:dmarck-reports@example.co.uk` (use a virtual)
- 

### Useful resources:

- https://www.linuxbabe.com/mail-server/setting-up-dkim-and-spf
- https://www.linuxbabe.com/mail-server/create-dmarc-record
- https://www.linuxbabe.com/mail-server/setup-basic-postfix-mail-sever-ubuntu 
- https://www.linuxbabe.com/mail-server/block-email-spam-check-header-body-with-postfix-spamassassin
