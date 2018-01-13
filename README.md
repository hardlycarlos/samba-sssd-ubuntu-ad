# samba-sssd-ubuntu-ad
Integrating Samba for file sharing on Ubuntu 16.04 into a Windows Active
Directory environment with SSSD authentication.

## The Problem
A small business with a heterogeneous network environment needed a file server
for user home directories. There are two Windows Server 2012 servers providing
Acitive Directory services in Windows Server 2008 mode. Other servers include a
Windows Server 2008 database server and a MacOS Sierra Server. Windows client
machines are all running Windows 10 Pro. All Apple clients are running MacOS
High Sierra. All clients and servers authenticate directly against Active
Directory.

The OS selected for the file server was Ubuntu 16.04.3 "Xenial Xerus" running
samba and System Security Services Daemon (SSSD). We also specifically wanted
to avoid winbind.

## About This Document
After an extensive search of Google, we were unable to find comprehensive
documentation that addressed all of the issues we encountered, at least one
that specifically pertained to Ubuntu. This document seeks to serve that
purpose. *However, these instructions assume a single forest, single domain.
Anything more will likely require significant modifications.*

## References
If you are curious about the documents/pages we primarily referenced, here they
are:
1. [Joining Ubuntu to an Active Directory Domain](https://blog.netnerds.net/2016/04/joining-ubuntu-to-an-active-directory-domain/)

2. [SSSD and Active Directory](https://help.ubuntu.com/lts/serverguide/sssd-ad.html)

3. [Integrating Ubuntu with Active Directory](https://noobient.com/post/132414642261/integrating-ubuntu-with-active-directory)

4. [Ubuntu 14.04 Active Directory Authentication](http://koo.fi/blog/2015/06/16/ubuntu-14-04-active-directory-authentication/)

5. [Adding AD domain groups to /etc/sudoers](https://derflounder.wordpress.com/2012/12/14/adding-ad-domain-groups-to-etcsudoers/)

6. [Add a Simple Samba File Server as a Domain Member](http://linuxtot.com/add-a-simple-samba-file-server-as-a-domain-member/)

7. [Red Hat Enterprise Linux 7, System Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-File_and_Print_Servers#setting_extended_acls)

## Variables
We understand that very often, people are just looking for a cut-and-paste
solution. At the same, it can be confusing to know what name to use where, etc.
To address both of those points, we have provided a list of variables with
examples and descriptions. If you enter your own values between the quotes and
then paste into bash, you should be able to pretty well cut-and-paste everything
else. If nothing else, this should resolve any ambiguity about what to actually
enter, captialization, etc.

We don't recommend simply cutting and pasting. We are going to try and
reasonably explain everything as we go, because we should all have a reasonable
understanding of why we are doing whatever it is that we are doing.

### Domain Administrator Username
This is the username for a user with sufficient privileges to add a new computer
to the domain. This user will generally be a member of the Domain Admins
Windows security group. Hopefully, it is not the default "Administrator" account,
which really should be disabled.
```
ADMIN_USER="administrator"
```

### Domain Administrator Group
This the name of a Windows group that is going to be granted permissions to
administer this server. Members of this group will be given sudo privileges.
```
DOMAIN_ADMINS="Domain Admins"
```

### Linux Machine Host Name
This is the desired short name for the Linux host.
```
MACHINE_NAME="Ubuntu"
```

### Domain Controllers Short Names
**Caution!** *The general recommendation is to have at least two domain
controllers when deploying Active Directory.* We used two, but you may have
one, three, or even more. Please pay attention to these two variables, and
modify accordingly, when they appear below.

```
DC1="DomainController1"
DC2="DomainController2"
```

### Legacy Domain Name
This is the legacy domain name, sometimes referred to as a pre-Windows 2000
domain name, a NetBIOS deomain, or domain short name. It should be all caps.
Usually, but not necessarily, if your domain name is example.com, then the
legacy domain name would be "EXAMPLE." Honestly, this may not even be
necessary, but since it is in the samba documentation, we are including it
(*see* [Setting up Samba as a Domain Member](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member)).

```
LDN="EXAMPLE"
```

### DNS Domain Name
This is self-explanatory, but it is the DNS name of your domain, e.g.
example.com. This should be in lower-case.

```
DOMAIN='example.com'
```

### Realm
Realm name, which will be used by Kerberos. This is your domain name in **upper-
case**.

```
DOMAIN='EXAMPLE.COM'
```

### Archive Folder
We are going to replace some default configuration files, but you might want
to be able to reference them later. Provide a path where these files can be
stored.

```
ARCHIVES="$HOME/original_configs"
```

## Networking: Static vs. Dynamic
You should be fine with a dynamically assigned IP address. Acitve Directory is
heavily dependent upon DNS. This of course assumes that your DHCP server
correctly updates the DNS server. Configuration of network settings is beyond
the scope of this guide, so if you need help, please review the official
documentation. That said, you should ensure that your DNS servers correctly
resolve the Active Directory records. The check is easy.

```
dig -t srv _ldap._tcp.$DOMAIN
```

The above command will return a list of all the IP addresses for all of your
domain controllers. If it doesn't, then you need to fix your DNS before
proceeding any further.

## Create Some Directories
Let's go ahead and create our directories.

```
# Folder to archive original configuration files.
mkdir -P $ARCHIVES

# Make a directory for samba
sudo mkdir -p /srv/samba/users
```


## Change the Hostname

There is no requirement for you to change the hostname. But this is probably
the time to do so, if you want to do so.

```
sudo hostnamectl set-hostname "$MACHINE_NAME\.$DOMAIN"
```


## Modify /etc/hosts
We really don't know why, but we ran accross quite a bit of advice to make
the following modification to /etc/hosts. Apparently, it has something to do
with properly registering the machine in DNS, but it doesn't make sense to us.
Nevertheless, we have chosen to follow the advice, because it doesn't hurt
anything, and especially if we have changed the name, it will allow our machine
to correctly address itself.

```
sudo sed -i \
-e "s|^127\.0\.1\.1.*|127\.0\.1\.1       $MACHINE_NAME\.$DOMAIN $MACHINE_NAME|" \
/etc/hosts
```

## Install Packages
We don't need to select much.

```
sudo apt update && sudo apt upgrade

sudo apt -y install \
acl \
krb5-user \
libwbclient-sssd \
ntp \
ntpdate \
samba \
smbclient \
sssd \
sssd-tools \
```

## Stop Services
We should go ahead and stop any of the services we just installed that are
running.

```
sudo service ntp stop
sudo service smbd stop
sudo service nmbd stop
```
You won't have to worry about stopping SSSD. It won't run if everything isn't
just so.


## Delete Samba Databases
The following commands delete the database files where they are
likely to be found. Ignore complaints that files cannot be found. That is a
good thing.

```
sudo rm /var/lib/samba/*.tdb
sudo rm /var/lib/samba/*.ldb
sudo rm /var/run/samba/*.tdb
sudo rm /var/run/samba/*.ldb
sudo rm /var/cache/samba/*.tdb
sudo rm /var/cache/samba/*.ldb
sudo rm /var/lib/private/samba/*.tdb
sudo rm /var/lib/private/samba/*.ldb
```











