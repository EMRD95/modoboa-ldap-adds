# modoboa-ldap-adds
### Tutorial for a local installation of modoboa with LDAP integration, Windows Server 2022 + Ubuntu 22.04 LTS Server.

(local test use) 



## Installation Modoba
Follow the installation instructions on [Modoboa's documentation](https://modoboa.readthedocs.io/en/latest/installation.html).

```bash
git clone https://github.com/modoboa/modoboa-installer
cd modoboa-installer
sudo ./run.py your.domain
```

Create a A record in your DNS server that points your email server to the domain it's associated to.

Add a domain in modoboa, gather the DNS rules necessary and set them up in your DNS server.

You should be able to send email with local accounts.

## LDAP setup

Create an Organisation Unit for your users, with a QueryUser account. Other Users should have an email address specified in their informations.

For LDAP authentication and synchronization, refer to the following resources:
- [LDAP Auth](https://modoboa.readthedocs.io/en/latest/configuration.html#ldap-auth)
- [Issue #114 for additional context](https://github.com/modoboa/modoboa-installer/issues/114)

Install necessary packages:

```bash
sudo apt-get update
sudo apt-get install libldap2-dev libsasl2-dev
```

Switch to modoboa user and activate the environment:

```bash
sudo su - modoboa
bash
source env/bin/activate
pip install python-ldap django-auth-ldap
```

Uncomment the ldap line in:

```bash
sudo nano /srv/modoboa/instance/instance/settings.py
```

![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/2d4d139b-6417-4d13-86dc-8759c786a0c3)

Add the MODOBOA_APPS ldapsync
[documentation](https://modoboa.readthedocs.io/en/latest/configuration.html#ldap-synchronization).

![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/224840ca-6f27-4ce2-ab33-3bd1921bbb6b)

## Set up Dovecot [If you want to be able to login with third party email services like Thunderbird or Nextcloud, users will be able to login but not to send or receive messages till they connected the first time to the modoboa webmail to create the modoboa account...]

```bash
sudo apt-get install dovecot-ldap
```

Configure Dovecot LDAP:

```bash
sudo nano /etc/dovecot/dovecot-ldap.conf.ext
```

Add the following configuration:

```
hosts           = 192.168.10.xx:389
ldap_version    = 3
auth_bind       = yes
dn              = queryuser
dnpass          = PASSWORD
base            = OU=modoba,OU=Units,DC=ad,DC=domain,DC=name
scope           = subtree
deref           = never
user_filter     = (&(userPrincipalName=%u)(objectClass=person)(!(userAccountControl=514)))
pass_filter     = (&(userPrincipalName=%u)(objectClass=person)(!(userAccountControl=514)))
pass_attrs      = userPassword=password
default_pass_scheme = CRYPT
user_attrs = =home=/srv/vmail/%Ld/%Ln,=mail=maildir:/srv/vmail/%Ld/%Ln
```

Configure LDAP in Dovecot's auth system:

```bash
sudo nano /etc/dovecot/conf.d/auth-ldap.conf.ext
```

Ensure the following is present:

```ini
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
 
userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
```


Uncomment !include auth-ldap.conf.ext
```bash
sudo nano +124 /etc/dovecot/conf.d/10-auth.conf
```

Set up the UID and GID
```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```
![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/b5f15324-bc0e-47ed-8652-21fa2b200323)


Verify LMTP service configuration in Dovecot, ensure the `service lmtp` section is correctly configured.

```bash
sudo nano /etc/dovecot/conf.d/10-master.conf
```

```bash
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   group = postfix
   mode = 0600
   user = postfix
  }
}
```


### Modoboa LDAP and Dovecot Synchronization

Configure LDAP in Modoboa's admin panel as specified, ensuring settings like server address, Active Directory flags, and search bases are correctly entered.


In domain.com/core/#parameters/ check 

Authentication type ? LDAP

Update password scheme at login ? No

Server address ? 192.168.10.xx

Active Directory ? Yes

Administrator groups ? OU=modoba,OU=Units,DC=ad,DC=domain,DC=name

Group type ? GroupOfNames

Groups search base ? OU=modoba,OU=Units,DC=ad,DC=domain,DC=name

Password attribute ? userPassword

Authentication method ? Search and bind

Bind DN ? CN=queryuser,OU=modoba,OU=Units,DC=ad,DC=domain,DC=name

Bind password ? PASSWORD

Users search base ? OU=modoba,OU=Units,DC=ad,DC=domain,DC=name

Search filter ? (mail=%(user)s)

Bind DN ? CN=queryuser,OU=modoba,OU=Units,DC=ad,DC=domain,DC=name

Bind password ? PASSWORD

Enable export to LDAP ? No

Enable import from LDAP ? Yes

Enable Dovecot LDAP sync ? No


Restart
```bash
sudo systemctl restart dovecot
```

or

```bash
sudo reboot
```

Make sure to replace placeholders like `your.domain`, `192.168.10.xx`, and `PASSWORD` with your actual data.

ACME domain setup with cloudflare API
```bash
sudo su
#https://github.com/acmesh-official/acme.sh/wiki/sudo
 
optional, for server edition
# apt-get install socat
 
curl https://get.acme.sh | sh -s email=mail@mail.com

source  ~/.bashrc
 
acme.sh --help
 
acme.sh --set-default-ca --server letsencrypt

export CF_Token="TOKEN DNS ZONE"
export CF_Email="mail@mail.com"

acme.sh --issue --dns dns_cf -d domain.com
 
Auto import

acme.sh --install-cert -d domain.com \
--key-file /etc/ssl/private/domain.com.key \
--fullchain-file /etc/ssl/certs/domain.com.cert \
--reloadcmd "service nginx force-reload"

```
Uncomment the sll line and restart dovecot
```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```
![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/398c7242-e203-488f-85c2-ff4bc78fa160)


You can also check if the path for the newly generated certificate is in place
For the web part, you'll find the configuration in /etc/nginx/conf.d/.
For the smtp part, it will be in /etc/postfix/main.cf



