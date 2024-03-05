# Debian - Active Directory Member Server Setup
Scripts and configuration files needed to set up a Debian desktop in an Active Directory Domain.

Reference links:

* https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member

Create a machine in VirtualBox:

* Name: Debian
* Type: Linux
* Version: Debian (64-bit)
* CPUs: 2
* RAM: 4096 MB
* Video Memory: 64 MB
* Virtual HD: 50.00 GB
* HD Type: VDI, dynamically allocated
* Set "Devices | Shared Clipboard" to Bidirectional

Use these Network settings for all machines in VirtualBox:

* Adapter 1: Enabled
  * Attached to: NAT Network
  * Name: NatNetwork  (10.0.2.0/24 – DHCP & IPv6 disabled)
* Adapter 2: Disabled  <-- Only needed for machines without a desktop.
  * Attached to: Host-only Adapter
  * Name: VirtualBox Host-Only Ethernet Adapter (192.168.56.0/24 – DHCP & IPv6 disabled)

Download the Debian image. Boot from it to begin the installation.

* Hostname: Debian
* Domain: samdom.example.com
* Enter the desired user name and password for the admin (sudo) account.
* Make your disk partition selections and write changes to disk.
* Software selection: Debian desktop environment, Xfce (my pick), SSH server, standard system utilities.
* Install the GRUB boot loader on /dev/sda
* Finish the installation and reboot.

Login as the admin user and check for updates:
```
sudo apt update
```
If there are updated packages, run these commands. Otherwise skip to the next step.
```
sudo apt full-upgrade
sudo reboot
```
Login as the admin user and mount Guest Additions (Devices | Insert Guest Additions CD image).
Install the Linux Guest Additions
```
sudo apt install build-essential linux-headers-$(uname -r) -y
sudo <path to Guest Additions>/VBoxLinuxAdditions.run
sudo reboot
```
Login as the admin user and switch to root.
Clone git repository to download these instructions, scripts and configuration files:
```
sudo su -
git clone https://github.com/TedMichalik/Debian.git
```
## Install software and copy config files to their proper location:
```
Debian/CopyFiles
```
Change the default UMASK in the /etc/login.defs file (Done with CopyFiles):
```
UMASK 002
```
Sync time with the AD DC by adding this line to the /etc/systemd/timesyncd.conf file:
```
NTP=DC1.samdom.example.com
```
Install Samba and packages needed to integrate into the domain (Done with CopyFiles).
```
apt install -y samba winbind libpam-winbind libnss-winbind libpam-krb5 krb5-config krb5-user
```
Also install some utility programs (Done with CopyFiles):
```
apt install -y net-tools
```
Stop samba services, backup configuration file and create a new one (Done with CopyFiles):
```
systemctl stop smbd nmbd winbind
mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
nano /etc/samba/smb.conf
```
Add these lines to the new **/etc/samba/smb.conf** (Done with CopyFiles)
```
[global]
        realm = SAMDOM.EXAMPLE.COM
        security = ADS
        server role = member server
        winbind enum groups = Yes
        winbind enum users = Yes
        winbind offline logon = Yes
        winbind use default domain = Yes
        workgroup = SAMDOM
        idmap config * : backend = tdb
        idmap config * : range = 3000-9999
        idmap config samdom : backend = ad
        idmap config samdom : range = 10000-999999
        idmap config samdom : unix_nss_info = yes
        map acl inherit = Yes
        vfs objects = acl_xattr
        server min protocol = NT1
        client min protocol = NT1

[homes]
        browseable = No
        comment = Home Directories
        create mask = 0644
        directory mask = 02755
        read only = No

[Public]
        comment = Shared Files
        create mask = 0664
        directory mask = 02775
        guest ok = Yes
        path = /opt/Public
        read only = No
```
Edit the Kerberos configuration file**/etc/krb5.conf**. It just needs these lines (Done with CopyFiles):
```
[libdefaults]
    default_realm = SAMDOM.EXAMPLE.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
```
Give sudo access to members of “domain admins” (Done with CopyFiles):
```
echo "%domain\ admins ALL=(ALL) ALL" > /etc/sudoers.d/SAMDOM
chmod 0440 /etc/sudoers.d/SAMDOM
```
Test Kerberos authentication against an AD administrative account and list the ticket by issuing the commands:
```
kinit administrator
klist
```
Join the Domain, and Restart Samba
```
samba-tool domain join samdom.example.com MEMBER -U administrator
systemctl start smbd nmbd winbind
```
Enable "Create home directory on login"
```
pam-auth-update
```
Create the Public folder:
```
mkdir /opt/Public
chgrp 'Domain Users' /opt/Public  ##If errors, check for "winbind" at end of passwd and group in /etc/nsswitch
chmod 2775 /opt/Public
```
Reboot to make sure everything works:
```
reboot
```
## Test the Member Server
Login as the admin user. Verify the Public share is present (it will fail the first time):
```
smbclient -L localhost -U%
```
Verify the DNS configuration works correctly:
```
host -t SRV _ldap._tcp.samdom.example.com.
host -t SRV _kerberos._udp.samdom.example.com.
host -t A dc1.samdom.example.com.
```
Verify the domain users are shown by both commands:
```
wbinfo -u
getent passwd
```
Verify the domain groups are shown by both commands:
```
wbinfo -g
getent group
```
Logout the admin user. You should now be able to login a domain user.

Verify that a home directory was created and that umask is 002:
```
pwd
umask
```
Verify the domain ownership on a test file:
```
touch /opt/Public/testfile
ls -l /opt/Public/
```
