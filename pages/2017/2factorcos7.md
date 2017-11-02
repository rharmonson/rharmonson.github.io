---
title: Two Factor Authentication using FreeRADIUS with SSSD and Google Authenticator on CentOS 7
published: true
tags: [linux, centos, service, sysadmin, security]
keywords: authentication, freeradius, radius, sssd, freeipa,
last_updated: October 31, 2017
summary: "Build a open source (*free*) two-factor authentication solution using FreeRADIUS, SSSD, and Google Authenticator."
sidebar: 2factorcos7_sidebar
permalink: 2factorcos7.html
folder: /pages/2017
toc: false
commentIssueId: 6
---

## Objective

The primary objective of this article is to provide an open source (free) two-factor authentication solution for use with network devices and VPN services.

## Architecture Overview

`[Place new system diagram and data flow HERE]`

*old system diagram*

![Architecture Diagram](https://cloud.githubusercontent.com/assets/7966126/7761484/365419ae-ffdb-11e4-9bad-4aa043d2c427.jpg)

## Solution Components

* CentOS 7 or Red Hat Enterprise Linux 7.4.1708
* Pluggable Authentication Module (PAM)
* FreeRADIUS 3.0.13
* System Security Services Daemon (SSSD)
* Google Authenticator 1.04
* Google Authenticator App
* Network Access Server (NAS) [RADIUS client, e.g. VPN service]

I will be using SSSD against FreeIPA (IPA) where IPA is "Identity, Policy, and Audit" which is the upstream project for Red Hat Identity Manager (IdM).

{% include note.html content="<br/>
Previously, I documented the use of SSSD against Microsoft Active Directory and you can find it at the URL given below.<br/>
<br/>
https://github.com/rharmonson/richtech/wiki/CentOS-7-Minimal-&-Two-factor-Authentication-using-FreeRADIUS-3,-SSSD-1.12,-&-Google-Authenticator<br/>
<br/>
I will link the appropriate section in the article above for Active Directory administrators using this article to install."
%}

## Firewall Ports

NAS to RADIUS

* UDP 1812; RADIUS

RADIUS to IPA

* UDP 53; DNS
* UDP 123; NTP
* TCP 88; Kerberos
* TCP 636; LDAPS

## Installation Overview

1. Base CentOS 7 installation
1. Prerequisites
1. FreeRADIUS Installation
1. Test FreeRADIUS local Unix account
1. SSSD (IPA or AD)
1. Test FreeRADIUS using SSSD account
1. Google Authenticator
1. Configure PAM
1. Test FreeRADIUS using SSSD & Google Authentication
1. Configure your NAS (not covered)
1. Test FreeRADIUS & NAS
1. Tidy Up! (optional)

## Prerequisites

### CentOS 7

Complete an installation of CentOS 7.4.1708 for building the FreeRADIUS service. You may use my guide found at the URL below, but if not, adjust the installation instructions to fit your CentOS build, e.g. firewalld versus iptables.

Note the instructions to configure Extra Packages for Enterprise Linux (EPEL) repository is optional.


https://rharmonson.github.io/cos7inst.html


### FreeIPA (IPA)

IPA is utilized by FreeRADIUS to authenticate users. This article does not describe the installation and configuration of IPA, however, my guide for installing an IPA Master and Replica can be found here:


https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Identity-Management-with-FreeIPA-Server-4.4-on-CentOS-7.3.1611


### Time

*Consistent and accurate time* is a key requirement for operations of the proposed solution. The FreeRADIUS host will be utilizing SSSD integration with IPA and as such both must have the correct time. In addition, Google Authenticator service and the device with the Google Authenticator App must have consistent time as well if using time based One Time Passwords (OTP). If problems occur during this tutorial with either SSSD or Google Authenticator, verify the time is correct.

### SELinux

Use `getenforce` to check the current SELinux setting. Use `sudo setenforce 0` to set to permissive for the current session. Execute the command below to update SELinux's configuration to use permissive on boot.

```
[randomusr@radsvc ~]$ sudo sed -i 's/=enforcing/=permissive/g' /etc/selinux/config
```

### Firewall

If using iptables-services as describe in my CentOS 7 Install Guide, create or update the existing firewall script to include UDP:1812 (authentication). RADIUS may use UDP or TCP protocols, but since UDP was the original protocol, most NAS will use it. Port 1813 is used by RADIUS for accounting. Enable it if applicable for your implementation.

```
#!/bin/bash
# FreeRADIUS IPv4 Polcies

#Flush current policies
iptables -F

# Set default chain policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow established sessions to receive traffic
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Accept on localhost
iptables -A INPUT -i lo -j ACCEPT

#ICMP Echo (OPTIONAL)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited

# Accept incoming SSH
iptables -I INPUT -p tcp -m conntrack --ctstate NEW --dport 22 -j ACCEPT

# FreeRADIUS: Authentication = 1812 / Accounting = 1813
# Your NAS will generally dictate port and protocol
iptables -I INPUT -p udp -m conntrack --ctstate NEW --dport 1812 -j ACCEPT
#iptables -I INPUT -p udp -m conntrack --ctstate NEW --dport 1813 -j ACCEPT

# Save Changes
service iptables save

# Service
systemctl restart iptables
systemctl status iptables
```

If using firewalld, use the section below to permit access for RADIUS.

```
[randomusr@radsvc ~]$ sudo firewall-cmd --list-service --zone=public
[randomusr@radsvc ~]$ sudo firewall-cmd --permanent --zone=public --add-service=radius
[randomusr@radsvc ~]$ sudofirewall-cmd --reload
[randomusr@radsvc ~]$ sudo firewall-cmd --list-service --zone=public
```

## FreeRADIUS

### Install FreeRADIUS

```
[randomusr@radsvc ~]$ sudo yum -y install freeradius freeradius-utils
```

Results

```
================================================================================
 Package                     Arch       Version               Repository   Size
================================================================================
Installing:
 freeradius                  x86_64     3.0.13-8.el7_4        updates     1.1 M
 freeradius-utils            x86_64     3.0.13-8.el7_4        updates     221 k
Installing for dependencies:
 apr                         x86_64     1.4.8-3.el7           base        103 k
 apr-util                    x86_64     1.5.2-6.el7           base         92 k
 boost-system                x86_64     1.53.0-27.el7         base         40 k
 boost-thread                x86_64     1.53.0-27.el7         base         57 k
 log4cxx                     x86_64     0.10.0-16.el7         base        452 k
 perl                        x86_64     4:5.16.3-292.el7      base        8.0 M
 perl-Carp                   noarch     1.26-244.el7          base         19 k
 perl-Compress-Raw-Bzip2     x86_64     2.061-3.el7           base         32 k
 perl-Compress-Raw-Zlib      x86_64     1:2.061-4.el7         base         57 k
 perl-DBI                    x86_64     1.627-4.el7           base        802 k
 perl-Data-Dumper            x86_64     2.145-3.el7           base         47 k
 perl-Encode                 x86_64     2.51-7.el7            base        1.5 M
 perl-Exporter               noarch     5.68-3.el7            base         28 k
 perl-File-Path              noarch     2.09-2.el7            base         26 k
 perl-File-Temp              noarch     0.23.01-3.el7         base         56 k
 perl-Filter                 x86_64     1.49-3.el7            base         76 k
 perl-Getopt-Long            noarch     2.40-2.el7            base         56 k
 perl-HTTP-Tiny              noarch     0.033-3.el7           base         38 k
 perl-IO-Compress            noarch     2.061-2.el7           base        260 k
 perl-Net-Daemon             noarch     0.48-5.el7            base         51 k
 perl-PathTools              x86_64     3.40-5.el7            base         82 k
 perl-PlRPC                  noarch     0.2020-14.el7         base         36 k
 perl-Pod-Escapes            noarch     1:1.04-292.el7        base         51 k
 perl-Pod-Perldoc            noarch     3.20-4.el7            base         87 k
 perl-Pod-Simple             noarch     1:3.28-4.el7          base        216 k
 perl-Pod-Usage              noarch     1.63-3.el7            base         27 k
 perl-Scalar-List-Utils      x86_64     1.27-248.el7          base         36 k
 perl-Socket                 x86_64     2.010-4.el7           base         49 k
 perl-Storable               x86_64     2.45-3.el7            base         77 k
 perl-Text-ParseWords        noarch     3.29-4.el7            base         14 k
 perl-Time-HiRes             x86_64     4:1.9725-3.el7        base         45 k
 perl-Time-Local             noarch     1.2300-2.el7          base         24 k
 perl-constant               noarch     1.27-2.el7            base         19 k
 perl-libs                   x86_64     4:5.16.3-292.el7      base        688 k
 perl-macros                 x86_64     4:5.16.3-292.el7      base         43 k
 perl-parent                 noarch     1:0.225-244.el7       base         12 k
 perl-podlators              noarch     2.5.1-3.el7           base        112 k
 perl-threads                x86_64     1.87-4.el7            base         49 k
 perl-threads-shared         x86_64     1.43-6.el7            base         39 k
 tncfhh                      x86_64     0.8.3-16.el7          base        680 k
 tncfhh-libs                 x86_64     0.8.3-16.el7          base        160 k
 tncfhh-utils                x86_64     0.8.3-16.el7          base         33 k
 xerces-c                    x86_64     3.1.1-8.el7_2         base        878 k

Transaction Summary
================================================================================
Install  2 Packages (+43 Dependent packages)

Total download size: 16 M
Installed size: 51 M
Is this ok [y/d/N]:
```

### Configure FreeRADIUS

{% include important.html content="<br/>
This solution's use of FreeRADIUS must run as root to access the .google_authenticator in user home directories."
%}

Edit radiusd.conf

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/radiusd.conf
```

Update `user` and `group` from

```
user = radiusd
group = radiusd
```

to

```
user = root
group = root
```

Edit sites-enabled/default

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/sites-enabled/default
```

Update the `pam` under "Pluggable Authentication Modules" from

```
#       pam
```

to

```
        pam
```

Enable the pam module

```
[randomusr@radsvc ~]$ sudo ln -s /etc/raddb/mods-available/pam /etc/raddb/mods-enabled/pam
```

Results

```
[randomusr@radsvc ~]$ sudo ls -l /etc/raddb/mods-enabled/pam
lrwxrwxrwx. 1 root root 29 Oct 28 21:17 /etc/raddb/mods-enabled/pam -> /etc/raddb/mods-available/pam
```

Configure clients.conf

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/clients.conf
```

Add the following above "client localhost {" where the IP address is the NAS or RADIUS client, e.g. VPN service.

```
client myNAS {
        ipaddr = 192.168.21.1
        secret = myNASpasswd
        require_message_authenticator = no
        nas_type = other
}
```

Configure 'users'

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/users
```

Update DEFAULT Group from

```
#DEFAULT        Group == "disabled", Auth-Type := Reject
#               Reply-Message = "Your account has been disabled."
#
```

to

```
DEFAULT Group == "disabled", Auth-Type := Reject
                Reply-Message = "Your account has been disabled."

DEFAULT Auth-Type := PAM
```

### Test FreeRADIUS with an UNIX account

Start radiusd in debug mode--use `ctrl+c` to end the radiusd sesssion when done.

```
[root@radsvc~]# radiusd -X
```

Results

```
..
[ lines of configuration details]
}
Listening on auth address * port 1812 bound to server default
Listening on acct address * port 1813 bound to server default
Listening on auth address :: port 1812 bound to server default
Listening on acct address :: port 1813 bound to server default
Listening on auth address 127.0.0.1 port 18120 bound to server inner-tunnel
Listening on proxy address * port 45094
Listening on proxy address :: port 35184
Ready to process requests
```

Open a additional terminal or SSH session and create a test user

```
[randomusr@radsvc ~]$ sudo useradd raduser
[randomusr@radsvc ~]$ sudo passwd raduser
Changing password for user raduser.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

Test using radtest from radiusd-util package and the local unix account, raduser.

```
[randomusr@radsvc ~]$ radtest raduser Password1 localhost 0 testing123
```

Results

```
Sent Access-Request Id 220 from 0.0.0.0:33872 to 127.0.0.1:1812 length 77
        User-Name = "raduser"
        User-Password = "Password1"
        NAS-IP-Address = 192.168.21.1
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "Password1"
Received Access-Accept Id 220 from 127.0.0.1:1812 to 0.0.0.0:0 length 20
```

Note the last line has "Received Access-Accept" which means success!

The terminal with the RADIUS daemon will show

```
(0) Received Access-Request Id 220 from 127.0.0.1:33872 to 127.0.0.1:1812 length 77
(0)   User-Name = "raduser"
(0)   User-Password = "Password1"
(0)   NAS-IP-Address = 192.168.21.1
(0)   NAS-Port = 0
(0)   Message-Authenticator = 0x939ba699b4d49n2c4bb04df5lk202498
(0) # Executing section authorize from file /etc/raddb/sites-enabled/default
(0)   authorize {
(0)     policy filter_username {
(0)       if (&User-Name) {
(0)       if (&User-Name)  -> TRUE
(0)       if (&User-Name)  {
(0)         if (&User-Name =~ / /) {
(0)         if (&User-Name =~ / /)  -> FALSE
(0)         if (&User-Name =~ /@[^@]*@/ ) {
(0)         if (&User-Name =~ /@[^@]*@/ )  -> FALSE
(0)         if (&User-Name =~ /\.\./ ) {
(0)         if (&User-Name =~ /\.\./ )  -> FALSE
(0)         if ((&User-Name =~ /@/) && (&User-Name !~ /@(.+)\.(.+)$/))  {
(0)         if ((&User-Name =~ /@/) && (&User-Name !~ /@(.+)\.(.+)$/))   -> FALSE
(0)         if (&User-Name =~ /\.$/)  {
(0)         if (&User-Name =~ /\.$/)   -> FALSE
(0)         if (&User-Name =~ /@\./)  {
(0)         if (&User-Name =~ /@\./)   -> FALSE
(0)       } # if (&User-Name)  = notfound
(0)     } # policy filter_username = notfound
(0)     [preprocess] = ok
(0)     [chap] = noop
(0)     [mschap] = noop
(0)     [digest] = noop
(0) suffix: Checking for suffix after "@"
(0) suffix: No '@' in User-Name = "raduser", looking up realm NULL
(0) suffix: No such realm "NULL"
(0)     [suffix] = noop
(0) eap: No EAP-Message, not doing EAP
(0)     [eap] = noop
(0) files: Failed resolving GID: No error
(0) files: users: Matched entry DEFAULT at line 67
(0)     [files] = ok
(0)     [expiration] = noop
(0)     [logintime] = noop
(0) pap: WARNING: No "known good" password found for the user.  Not setting Auth-Type
(0) pap: WARNING: Authentication will fail unless a "known good" password is available
(0)     [pap] = noop
(0)   } # authorize = ok
(0) Found Auth-Type = pam
(0) # Executing group from file /etc/raddb/sites-enabled/default
(0)   authenticate {
(0) pam: Using pamauth string "radiusd" for pam.conf lookup
(0) pam: Authentication succeeded
(0)     [pam] = ok
(0)   } # authenticate = ok
(0) # Executing section post-auth from file /etc/raddb/sites-enabled/default
(0)   post-auth {
(0)     update {
(0)       No attributes updated
(0)     } # update = noop
(0)     [exec] = noop
(0)     policy remove_reply_message_if_eap {
(0)       if (&reply:EAP-Message && &reply:Reply-Message) {
(0)       if (&reply:EAP-Message && &reply:Reply-Message)  -> FALSE
(0)       else {
(0)         [noop] = noop
(0)       } # else = noop
(0)     } # policy remove_reply_message_if_eap = noop
(0)   } # post-auth = noop
(0) Sent Access-Accept Id 220 from 127.0.0.1:1812 to 127.0.0.1:33872 length 0
(0) Finished request
Waking up in 4.9 seconds.
(0) Cleaning up request packet ID 220 with timestamp +379
Ready to process requests
```

## SSSD (IPA or AD)

IPA provides many services including DNS and NTP services. Configure the RADIUS host to use DNS and NTP from the IPA Master and Replicas.

* Master: 	ipa1.subdomain.mydomain.net;	192.168.3.1
* Replica:	ipa2.subdomain.mydomain.net;	192.168.3.2

{% include note.html content="<br/>
If using Microsoft Active Directory, use the link below to jump to my prior article describing its integration.<br/>
<br/>
https://github.com/rharmonson/richtech/wiki/CentOS-7-Minimal-&-Two-factor-Authentication-using-FreeRADIUS-3,-SSSD-1.12,-&-Google-Authenticator#sssd"
%}

### DNS

Update `resolv.conf` to use IPA for name resolution.

```
[randomusr@radsvc ~]$ sudo vi /etc/resolv.conf
```

Results

```
search subdomain.mydomain.net
nameserver 192.168.3.1
nameserver 192.168.3.2
```

### NTP

I use `ntp` versus the new default `chrony` for time services on CentOS 7 specifically because of challenges with chrony and the IPA client installer in the past. In theory it is unnecessary to install ntp and configure it prior joining the FreeRADIUS host to IPA using the `--force-ntpd` for it will install the ntp package and configure ntp.conf. In practice I setup ntp prior to using ipa-client to avoid time issues and Kerberos authentication failures during IPA client installation.

Remove chrony

```
[randomusr@radsvc ~]$ sudo systemctl stop chronyd && sudo systemctl disable chronyd
[randomusr@radsvc ~]$ sudo yum -y remove chrony
```

Install ntp

```
[randomusr@radsvc ~]$ sudo yum -y install ntp
```

Update `ntp.conf`

```
[randomusr@radsvc ~]$ sudo vi /etc/ntp.conf
```

from

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
```

to

```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server ipa1.subdomain.mydomain.net
server ipa2.subdomain.mydomain.net

#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
```

Start ntp

```
[randomusr@radsvc ~]$ sudo systemctl enable ntpd && sudo systemctl start ntpd
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.
```

Verify ntp using `ntpq` and `ntpstat`.

```
[randomusr@radsvc ~]$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ipa1.subdomain.ha 192.168.10.53  3 u   26   64    3    0.304   -0.370   0.180
+ipa2.subdomain.ha 192.168.10.52  3 u   24   64    3    0.299    0.343   0.178

[randomusr@radsvc ~]$ ntpstat
synchronised to NTP server (192.168.3.1) at stratum 4
   time correct to within 281 ms
   polling server every 64 s
```

### Client Package

To use IPA, first install the `ipa-client` and its dependencies.

```
[randomusr@radsvc ~]$ sudo yum -y install ipa-client
```

### Join

Time to join the FreeRADIUS host to the IPA Realm.

```
[randomusr@radsvc ~]$ sudo ipa-client-install --force-ntpd --enable-dns-updates --mkhomedir
Discovery was successful!
Client hostname: radsvc.subdomain.mydomain.net
Realm: subdomain.mydomain.NET
DNS Domain: subdomain.mydomain.net
IPA Server: ipa1.subdomain.mydomain.net
BaseDN: dc=subdomain,dc=mydomain,dc=net

Continue to configure the system with these values? [no]: yes
```

Results

```
Synchronizing time with KDC...
Attempting to sync time using ntpd.  Will timeout after 15 seconds
User authorized to enroll computers: admin
Password for admin@subdomain.mydomain.NET:
Successfully retrieved CA cert
    Subject:     CN=CA Signing Certificate,OU=pki-tomcat,O=subdomain.mydomain.net Security Domain
    Issuer:      CN=CA Signing Certificate,OU=pki-tomcat,O=subdomain.mydomain.net Security Domain
    Valid From:  2017-03-09 21:43:58
    Valid Until: 2037-03-09 20:43:58

    Subject:     CN=Certificate Authority,O=subdomain.mydomain.NET
    Issuer:      CN=CA Signing Certificate,OU=pki-tomcat,O=subdomain.mydomain.net Security Domain
    Valid From:  2017-03-09 23:31:44
    Valid Until: 2027-03-09 23:31:44

Enrolled in IPA realm subdomain.mydomain.NET
Created /etc/ipa/default.conf
New SSSD config will be created
Configured sudoers in /etc/nsswitch.conf
Configured /etc/sssd/sssd.conf
Configured /etc/krb5.conf for IPA realm subdomain.mydomain.NET
trying https://ipa2.subdomain.mydomain.net/ipa/json
[try 1]: Forwarding 'schema' to json server 'https://ipa2.subdomain.mydomain.net/ipa/json'
trying https://ipa2.subdomain.mydomain.net/ipa/session/json
[try 1]: Forwarding 'ping' to json server 'https://ipa2.subdomain.mydomain.net/ipa/session/json'
[try 1]: Forwarding 'ca_is_enabled' to json server 'https://ipa2.subdomain.mydomain.net/ipa/session/json'
Systemwide CA database updated.
Hostname (radsvc.subdomain.mydomain.net) does not have A/AAAA record.
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ed25519_key.pub
[try 1]: Forwarding 'host_mod' to json server 'https://ipa2.subdomain.mydomain.net/ipa/session/json'
SSSD enabled
Configured /etc/openldap/ldap.conf
NTP enabled
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config
Configuring subdomain.mydomain.net as NIS domain.
Client configuration complete.
The ipa-client-install command was successful
```

After the successful installation, reboot the host.

### IPA Group & HBAC

Login to the IPA portal and verify the FreeRADIUS host is found under Identity --> Hosts. Complete the following:

1. Create group "vpnusers" using Identity --> Groups
1. Select "vpnuser" and the desired users to the group
1. Create policy "vpnuser_policy"  using Policy --> Host Based Access Control
1. Select Who --> User Groups and add "vpnusers"
1. Select Accessing --> Hosts and add "radsvc.subdomain.mydomain.net"
1. Select Via Service --> Services and add "login"
1. Save

### Test IPA

Using FreeRADIUS, login via terminal or SSH using an account in vpnusers. Note there is no need to specify the IPA realm.

```
CentOS Linux 7 (Core)
Kernel 3.10.0-693.5.2.el7.x86_64 on an x86_64

radsvc login: john
Password:
Creating home directory for john.
[john@radsvc ~]$
```

### Test FreeRADIUS with a SSSD account

Start FreeRADIUS in debug mode

```
[randomusr@radsvc ~]$ sudo radiusd -X
```

Open another shell and use the `radtest` utility and use a user that is a member of the group vpnusers.

```
[root@radsvc~]# radtest john Password1 localhost 0 testing123
```

Results should contain `Access-Accept` otherwise, backup and check your work.

```
[randomusr@radsvc ~]$ sudo radtest john 'Password1' localhost 0 testing123
Sent Access-Request Id 205 from 0.0.0.0:39112 to 127.0.0.1:1812 length 77
        User-Name = "john"
        User-Password = "Password1"
        NAS-IP-Address = 192.168.21.1
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "Password1"
Received Access-Accept Id 205 from 127.0.0.1:1812 to 0.0.0.0:0 length 20
```

and from radiusd

```
(3) Sent Access-Accept Id 205 from 127.0.0.1:1812 to 127.0.0.1:39112 length 0
```

Use 'ctrl+c' to kill the radiusd daemon.

## Google Authenticator

### Authenticator Package

Install the Google Authenticator package.

```
[randomusr@radsvc ~]$ sudo yum -y install google-authenticator
```

Results

```
================================================================================
 Package                     Arch          Version            Repository   Size
================================================================================
Installing:
 google-authenticator        x86_64        1.04-1.el7         epel         48 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 48 k
Installed size: 97 k
Is this ok [y/d/N]:
```

### Setup User

On-board the user using su and google-authenticator command.

```
[root@radsvc~]# su - john
Last login: Tue May 12 20:27:41 EDT 2015 from 172.16.1.1 on pts/1
[john@radsvc~]$ google-authenticator
```

Responding with `y` to queries results with

```
Do you want authentication tokens to be time-based (y/n) y
Warning: pasting the following URL into your browser exposes the OTP secret to Google:
  https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/john@mydomain.com@radsvc.mydomain.com%3Fsecret%3[reallylongsecretandmorestuff]

[QR image]

Your new secret key is: FRL4H7J4OOCY4QGA
Your verification code is 131105
Your emergency scratch codes are:
  86104574
  16057979
  31558510
  31258658
  14698995

Do you want me to update your "/home/john/.google_authenticator" file? (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) y

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```

The secret key is required to configure the Google Authenticator App so note and secure it in a safe place. The emergency scratch codes are evil but great for testing so copy/paste for initial testing.

## PAM

The `/etc/pam.d/radiusd` file needs to be configured to utilize both SSSD and Google Authenticator.

Update the radiusd file.

```
[randomusr@radsvc ~]$ sudo vi /etc/pam.d/radiusd
```

from

```
#%PAM-1.0
auth       include      password-auth
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
session    include      password-auth
```

to

```
#%PAM-1.0
auth       requisite    pam_google_authenticator.so forward_pass
auth       required     pam_sss.so use_first_pass
account    required     pam_nologin.so
account    include      password-auth
session    include      password-auth
```

{% include note.html content="<br/>
Alternative configuration is to remove 'pam_sss.so' and set 'auth required pam_google_authenticator.so' (no forward_pass or use_first_pass) as required. This results in being prompted for the token only. When integrating with applications, you would chain authentication for different authentication sources. For example, a VPN solution could query IPA or AD for user credentials, pass forward the user account and prompt for the token."
%}

```
#%PAM-1.0
auth       required     pam_google_authenticator.so
account    required     pam_nologin.so
account    include      password-auth
session    include      password-auth
```

## Test FreeRADIUS with SSSD & Google Authenticator

Lauch `sudo radiusd -X` and connect to another shell. In the other shell, use the radtest utility by providing a user within the vpnusers group and the account password followed by an Google Authenticator emergency scratch code. If your password has special characters, use `'`password`'`.

```
[randomusr@radsvc ~]$ sudo radtest <username> `[<ipa_account_password>+<google_authenticator_otp_token>]` localhost 0 testing123
```

Example

```
[randomusr@radsvc ~]$ sudo radtest john 'Password186104574' localhost 0 testing123
```

Results

```
[randomusr@radsvc ~]$ sudo radtest john 'Password186104574' localhost 0 testing123
Sent Access-Request Id 134 from 0.0.0.0:45970 to 127.0.0.1:1812 length 77
        User-Name = "john"
        User-Password = "Password186104574"
        NAS-IP-Address = 192.168.21.1
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "Password186104574"
Received Access-Accept Id 134 from 127.0.0.1:1812 to 0.0.0.0:0 length 20
```

The corresponding service response.

```
(0) Sent Access-Accept Id 134 from 127.0.0.1:1812 to 127.0.0.1:45970 length 0
```

Use 'ctrl+c' to kill the radiusd process.


## Test Google Authenticator App

Obtain Google Authenticator App for your mobile device via Google Play Store and setup using your secret key, e.g. `FRL4H7J4OOCY4QGA`.

Repeat the test from the section above titled <i>Test FreeRADIUS with SSSD & Google Authenticator</i> but use the OTP code provided by the app not the emergency scratch code. The test should result with `Received Access-Accept`.

## radiusd Enable

Enable and start the radiusd service. You killed your prior instances of radiusd using ctrl+c for testing, right?

```
[randomusr@radsvc ~]$ sudo systemctl enable radiusd && sudo systemctl start radiusd
Created symlink from /etc/systemd/system/multi-user.target.wants/radiusd.service to /usr/lib/systemd/system/radiusd.service.
```

## Network Access Server

The Network Access Server (NAS) can be any network device that supports the RADIUS protocol.

### Log-in process

1. For user, enter `john`
1. For password, enter `[<ipa_account_password>+<google_authenticator_otp_token>]` or `Password1437247`

If all goes well, you should successfully login.

## Tidy Up!

At this point, we have completed the basic build. The next section "Tidy Up!" provides a number of additional suggestions, but are not requirements.

### Delete local test user

```
[randomusr@radsvc ~]$ sudo userdel -r raduser
```

### Update clients.conf

Update the /etc/raddb/clients.conf file with an appropriate password for all secrets including the `localhost` connection. For example, change the default localhost from "testing123" to a secret with 12 to 16 upper and lower case characters, numbers, and symbols.

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/clients.conf
```

### Enable FreeRADIUS Logs

By default, logs are not enabled.

Update the sites-enabled/default to log accounting information.

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/sites-enabled/default
```

authorize section from

```
[randomusr@radsvc ~]$ sudo vi /etc/raddb/sites-enabled/default

        #
        #  If you want to have a log of authentication requests,
        #  un-comment the following line, and the 'detail auth_log'
        #  section, above.
#       auth_log
```

to

```
        auth_log
```


post-auth section from

```
        #
        #  If you want to have a log of authentication replies,
        #  un-comment the following line, and the 'detail reply_log'
        #  section, above.
#       reply_log
```

to

```
        reply_log
```

There is a dependency on the /etc/raddb/mods-enabled/detail.log, however, by default no changes need to be made. After auth_log and reply_log changes are made, it just works.

Restart FreeRADIUS

```
[randomusr@radsvc ~]$ sudo systemctl restart radiusd
```

Login via your NAS and verify logs are generated.

```
[randomusr@radsvc ~]$ sudo ls /var/log/radius/radacct/
192.168.21.1
[randomusr@radsvc ~]$ sudo ls /var/log/radius/radacct/192.168.21.1/
auth-detail-20171020  reply-detail-20171020
```

## Done!
