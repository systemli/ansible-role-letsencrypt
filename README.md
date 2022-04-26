# Ansible role to obtain Let's Encrypt SSL certificates

![Integration](https://github.com/systemli/ansible-role-letsencrypt/workflows/Integration/badge.svg?branch=main)
[![Ansible Galaxy](http://img.shields.io/badge/ansible--galaxy-letsencrypt-blue.svg)](https://galaxy.ansible.com/systemli/letsencrypt/)

This role is meant to request SSL certificates from Let's Encrypt,
using the HTTP or the DNS challenge for their ACME API.

Features:
* Installs and configures certbot and the DNS challenge helper script
* Supports both the HTTP and the DNS challenge
  * For HTTP challenge, the authenticator plugins `apache`, `nginx`, `standalone`
    and `webroot` are supported
* The DNS challenge uses a dedicated zone for AMCE challenge tokens
  only, lowering the security risks of dynamic updates. The concept
  is explained [here](https://www.crc.id.au/using-centralised-management-with-lets-encrypt/)
* Restart of services at certificate renewal using post-hooks or custom
  post-hook command
* Permission control to certificates using a dedicated system group

It does the following:
* When letsencrypt_setup is True (the default) this role will:
  * Install certbot
  * Register an account at Let's Encrypt
  * Install required files/keys for the DNS challenge
  * Create the system group 'letsencrypt'

* When invoked with filled variable 'letsencrypt_cert':
  * Requests a SSL certificate via the Let's Encrypt ACME API, either
    using the HTTP challenge or using the DNS challenge
  * Optionally sets the post-hook for certificate renewals (to restart
    required services afterwards)
  * Optionally adds system users to the 'letsencrypt' system group to
    grant them read access to the SSL certificates and their private keys

## How it works (examples)

 * Installation of certbot
   ```ansible-playbook site.yml -l localhost -t letsencrypt```
 * Creation of a certificate via HTTP challenge and with `webroot`
   authenticator (restarting service 'apache2' at renewal):
   ```ansible-playbook site.yml -l localhost -t letsencrypt -e '{"letsencrypt_cert":{"name":"sub.example.org","domains":["sub.example.org"],"challenge":"http","http_auth":"webroot","webroot_path":"/var/www/sub.example.org","services":["apache2"]}}'```
 * Creation of a certificate via DNS challenge (granting read access to
   certs to user 'Debian-exim', restarting services 'exim4' and 'dovecot`
   at renewal):
   ```ansible-playbook site.yml -l localhost -t letsencrypt -e '{"letsencrypt_cert":{"name":"sub2","domains":["sub2.example.org","sub2.another.example.org"],"challenge":"dns","services":["dovecot","exim4"],"users":["Debian-exim"]}}'```
 * Creation of a certificate via HTTP challenge and with `standalone`
   authenticator (re-using same private key at renewal and running a
   custom post-hook script at renewal):
   ```ansible-playbook site.yml -l localhost -t letsencrypt -e '{"letsencrypt_cert":{"name":"sub3","domains":["sub3.example.org"],"challenge":"http","http_auth":"standalone","reuse_key":True,"post_hook":"/usr/local/bin/cert-post-hook.sh"}}'```

## Expected structure of variable `letsencrypt_cert`

The variable `letsencrypt_cert` is expected to be a dictionary:

```
letsencrypt_opts_extra: "--register-unsafely-without-email"
letsencrypt_cert:
  name: sub.example.org
  domains:
    - sub.example.org
  challenge: http
  http_auth: webroot
  webroot_path: /var/www/sub.example.org
  services:
    - apache2
```

or:

```
letsencrypt_cert:
  name: sub2
  domains:
    - sub2.example.org
    - sub2.another.example.org
  challenge: dns
  services:
    - dovecot
    - exim4
  users:
    - Debian-exim
```

or:

```
letsencrypt_cert:
  name: sub3
  domains:
    - sub3.example.org
  challenge: http
  http_auth: standalone
  reuse_key: True
  post_hook: "/usr/local/bin/cert-post-hook.sh"
```

```
letsencrypt_cert:
  name: sub3
  domains:
    - sub3.example.org
  challenge: http
  http_auth: standalone
  reuse_key: True
  deploy_hook: "/usr/local/bin/cert-post-hook.sh"
```

The dictionary supports the following keys:

* `name`: name of the certificate [optional]
* `domains`: list of domains for the certificate [required]
* `challenge`: 'http' or 'dns' [required]
  * for challenge 'http': `http_auth`: 'webroot', 'apache' or 'nginx' [optional, default 'webroot']
    * for http_auth 'webroot': `webroot_path` [optional, default '/var/www']
* `services`: list of services to be restarted in the post-hook [optional]
* `reuse_key`: Reuse same private key at certificate renewal. 'True' or 'False'
   (default 'False')
* `post_hook`: Custom post-hook to be executed after attempt to obtain/renew
  a certificate [optional]
* `deploy_hook`: Custom deploy-hook to be executed after a successful attempt
  to obtain/renew  a certificate [optional]
* `renew_hook`: Custom renew-hook to be executed once for each renewed
  certificate after certificate renewal [optional]
* `users`: list of users to be added to system group 'letsencrypt' [optional]

## General Preliminaries

The role takes care of installing certbot and requesting SSL certificates
using either the HTTP or the DNS challenge. It doesn't install or configure
the required infrastructure (i.e. the Apache webserver or a DNS server).

The role is tested with Ansible 2.2 only. No guaranties that it runs with
earlier versions of Ansible.

## The HTTP challenge

Requirements:
* The domain name(s) of the requested certificate has to point
  to the system
* For http_auth 'apache', Apache2 has to be installed (and configured)
  on the system
* For http_auth 'nginx', NGINX has to be installed (and configured)
  on the system

## The DNS challenge

Requirements:
* A DNS server with a dedicated zone, used for the ACME DNS challenge only.
  This zone has to allow dynamic DNS updates (NSUPDATE) for TXT records
  (see below).
* CNAME records for `_acme-challenge.sub.example.org` for all domain
  names(s) of the requested certificate have to point to
  `sub.example.org._le.example.org` (inside the dedicated zone for the
  ACME DNS challenge).
* The content of the DNS update key and private DNS update keys need to be
  available in the Ansible vars `letsencrypt_ddns_key` and
  `letsencrypt_ddns_privkey` (preferably inside a vault).

This role installs a helper script for the DNS challenge to
`/usr/local/bin/certbot-dns-hook.sh`. This script will add the validation
token to the TXT record at `sub.example.org._le.example.org` during the DNS
challenge and remove it afterwards.

### Wildcard support with the DNS challenge

Obtaining wildcard certificates should work out of the box via DNS challenge.

### Configuring bind9 for the DNS challenge
*(Another option would be to use the [acme-dns server](https://github.com/joohoi/acme-dns) for this)*

Generate a key for dynamic updates:

```
cd /etc/bind/keys
dnssec-keygen -a HMAC-SHA512 -b 512 -n USER _le.example.org_ddns_update
chown -R bind:bind /etc/bind/keys
```

Add the key to your bind config (e.g. at `/etc/bind/named.conf.options`):

```
key "_le.example.org_ddns_update" {
	algorithm hmac-sha512;
	secret "...";
};
```

Create the zone for dynamic updates:

```
$ORIGIN .
$TTL 86400	; 1 day
_le.example.org		IN SOA	ns1.example.org. postmaster.example.org. (
				2017061501 ; serial
				86400      ; refresh (1 day)
				3600       ; retry (1 hour)
				2419200    ; expire (4 weeks)
				86400      ; minimum (1 day)
				)
			NS	ns1.example.org.
			NS	ns2.example.org.
			TXT	"v=spf1 -all"
```

and configure it in your bind config (e.g. at `/etc/bind/named.conf.local`):

```
zone "_le.example.org" {
	type master;
	file "/etc/bind/zones/db._le.example.org";
	update-policy { grant _le.example.org_ddns_update wildcard *._le.example.org. TXT; };
};
```

Format for `/etc/letsencrypt/keys/ddns_update.key` (from bind)

```
key "<key>" {
    algorithm HMAC-SHA512;
    secret "<key>";
};
```

Format for `/etc/letsencrypt/keys/ddns_update.private`

```
Private-key-format: v1.3
Algorithm: 165 (HMAC_SHA512)
Key: <key>
Bits: AAA=
Created: 20181017144534
Publish: 20181017144534
Activate: 20181017144534
```

# Ansible variable defaults

```
# Set the email address associated with the Let's Encrypt account
letsencrypt_account_email: ""

# Default authenticator for the HTTP challenge ('webroot' or 'apache')
letsencrypt_http_auth: webroot

# Default webroot path for the authenticator 'webroot'
letsencrypt_webroot_path: /var/www

# Install the DNS challenge helper script and DNS update key
letsencrypt_dns_challenge: yes

# Settings for the dynamic DNS zone updates
#letsencrypt_ddns_server: ""
#letsencrypt_ddns_zone: ""
#letsencrypt_ddns_key: ""
#letsencrypt_ddns_privkey: ""

# Create system group 'letsencrypt' for access to certificates
letsencrypt_group: yes

# Reuse private key at certificate renewal?
letsencrypt_reuse_key: False

# Allow subset of names?
letsencrypt_subset_names: True

# Set global extra commandline options for certbot
letsencrypt_opts_extra: ""
```

## Testing

For testing purposes, variable `letsencrypt_test` can be set. If set to
True, the role will use Let's Encrypt test servers for account creation
and obtaining the certificate.

For developing and testing the role we use Molecule and Vagrant/Github Actions. On the local environment you can easily test the role with

```
molecule test
```

## License

This Ansible role is licensed under the GNU GPLv3.

## Author

Copyright 2017-2019 systemli.org (https://www.systemli.org/)
