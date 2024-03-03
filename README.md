# modoboa-ldap-adds
### Local installation of modoboa with LDAP integration, Windows Server 2022 + Ubuntu 22.04 LTS Server.

(home server, for an email server that can send emails to the internet, don't use a domestic IP address) 



## Installation Modoba
Follow the installation instructions on [Modoboa's documentation](https://modoboa.readthedocs.io/en/latest/installation.html).

```bash
git clone https://github.com/modoboa/modoboa-installer
cd modoboa-installer
sudo ./run.py your.domain
```

Create a A record in your DNS server that points your mail server to the domain it's associated to.

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
sudo nano /srv/modoboa/instance/settings.py
```

![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/2d4d139b-6417-4d13-86dc-8759c786a0c3)

Add the MODOBOA_APPS ldapsync
[documentation](https://modoboa.readthedocs.io/en/latest/configuration.html#ldap-synchronization).

![image](https://github.com/EMRD95/modoboa-ldap-adds/assets/114953576/224840ca-6f27-4ce2-ab33-3bd1921bbb6b)

### Set up Dovecot

Keep your current configuration during installation

```bash
sudo apt-get install dovecot-ldap dovecot-core dovecot-lmtpd
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
user_attrs      = =home=/var/vmail/vmail1/%Ld/%Ln/Maildir/,=mail=maildir:/var/vmail/vmail1/%Ld/%Ln/Maildir/
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

The vmail user should already be created.

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

Enable Dovecot LDAP sync ? Yes


Restart
```bash
sudo systemctl restart dovecot
```

Make sure to replace placeholders like `your.domain`, `192.168.10.xx`, and `PASSWORD` with your actual data.
