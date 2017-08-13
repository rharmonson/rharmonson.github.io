---
title: Two Factor Authentication using Google Authenticator on CentOS 7
published: true
tags: [linux centos service sysadmin security]
keywords: authentication radius sssd
last_updated: July 1, 2016
summary: "The purpose of this article is to article is to provide a *free* two-factor authentication solution for use with VPN solutions."
permalink: 2factorcos7.html
folder: mydoc
toc: false
CommentIssueId: 6
---

# CentOS 7 Minimal: Two-factor Authentication using FreeRADIUS 3, SSSD 1.12, & Google Authenticator

## Objective

The primary objective of this article is to provide a *free* two-factor authentication solution for use with VPN solutions.

## Prerequisites

Before beginning, you will need to complete a minimal installation of CentOS 7 build 1503 or RHEL 7.1 and `yum update`. You can use my [GUIDE](https://github.com/rharmonson/richtech/wiki/CentOS-7-Minimal-x86_64-Base-Installation-Guide) found at the URL below. Note the instructions to configure Extra Packages for Enterprise Linux (EPEL) repository is not a requirement and can be ignored.

`https://github.com/rharmonson/richtech/wiki/CentOS-7-Minimal-x86_64-Base-Installation-Guide`

In addition, *consistent and accurate time* is a key requirement for the operation of the proposed solution. The FreeRADIUS host will be utilizing SSSD integration with Active Directory and as such both must have the same time. In addition, Google Authenticator service and the device with the Google Authenticator App must have consistent time as well if using time based One Time Passwords (OTP). If problems occur during this tutorial with either SSSD or Google Authenticator, verify the time is correct.

## Architecture Overview

![Architecture Diagram](https://cloud.githubusercontent.com/assets/7966126/7761484/365419ae-ffdb-11e4-9bad-4aa043d2c427.jpg)

## FreeRADIUS Components

* CentOS 7 (1503) or Red Hat Enterprise Linux 7.1 Minimal
* FreeRADIUS
* System Security Services Daemon (SSSD)
* Google Authenticator Pam Library, Service, & APP
* Pluggable Authentication Module (PAM)

## Firewall Requirements

User to NAS

* TCP 443; SSL VPN

NAS to RADIUS

* UDP 1812; RADIUS

RADIUS to Active Directory

* UDP 53; DNS
* UDP 123; NTP
* TCP 88; Kerberos
* TCP 389; LDAP
* TCP 3268 Global Catalog

## Installation Overview

1. Base CentOS 7 installation
1. FreeRADIUS Installation
1. Test FreeRADIUS local Unix account
1. SSSD Installation
1. Test FreeRADIUS using SSSD account
1. Google Authentication Compile & Installation
1. Configure PAM
1. Configure SELinux
1. Test FreeRADIUS using SSSD & Google Authentication
1. Configure Firewall
1. Configure your NAS (not covered)
1. Test FreeRADIUS & NAS
1. Tidy Up! (optional)

## FreeRADIUS

### Install FreeRADIUS

```
[root@2fcosrad7 ~]# yum install freeradius freeradius-utils
```

Results with

```
================================================================================
 Package                      Arch        Version               Repository
                                                                           Size
================================================================================
Installing:
 freeradius                   x86_64      3.0.4-6.el7           base      985 k
 freeradius-utils             x86_64      3.0.4-6.el7           base      188 k
Installing for dependencies:
 apr                          x86_64      1.4.8-3.el7           base      103 k
 apr-util                     x86_64      1.5.2-6.el7           base       92 k
 boost-system                 x86_64      1.53.0-23.el7         base       39 k
 boost-thread                 x86_64      1.53.0-23.el7         base       56 k
 libtalloc                    x86_64      2.1.1-1.el7           base       30 k
 log4cxx                      x86_64      0.10.0-16.el7         base      452 k
 perl                         x86_64      4:5.16.3-285.el7      base      8.0 M
 perl-Carp                    noarch      1.26-244.el7          base       19 k
 perl-Compress-Raw-Bzip2      x86_64      2.061-3.el7           base       32 k
 perl-Compress-Raw-Zlib       x86_64      1:2.061-4.el7         base       57 k
 perl-DBI                     x86_64      1.627-4.el7           base      802 k
 perl-Data-Dumper             x86_64      2.145-3.el7           base       47 k
 perl-Encode                  x86_64      2.51-7.el7            base      1.5 M
 perl-Exporter                noarch      5.68-3.el7            base       28 k
 perl-File-Path               noarch      2.09-2.el7            base       26 k
 perl-File-Temp               noarch      0.23.01-3.el7         base       56 k
 perl-Filter                  x86_64      1.49-3.el7            base       76 k
 perl-Getopt-Long             noarch      2.40-2.el7            base       56 k
 perl-HTTP-Tiny               noarch      0.033-3.el7           base       38 k
 perl-IO-Compress             noarch      2.061-2.el7           base      260 k
 perl-Net-Daemon              noarch      0.48-5.el7            base       51 k
 perl-PathTools               x86_64      3.40-5.el7            base       82 k
 perl-PlRPC                   noarch      0.2020-14.el7         base       36 k
 perl-Pod-Escapes             noarch      1:1.04-285.el7        base       50 k
 perl-Pod-Perldoc             noarch      3.20-4.el7            base       87 k
 perl-Pod-Simple              noarch      1:3.28-4.el7          base      216 k
 perl-Pod-Usage               noarch      1.63-3.el7            base       27 k
 perl-Scalar-List-Utils       x86_64      1.27-248.el7          base       36 k
 perl-Socket                  x86_64      2.010-3.el7           base       49 k
 perl-Storable                x86_64      2.45-3.el7            base       77 k
 perl-Text-ParseWords         noarch      3.29-4.el7            base       14 k
 perl-Time-HiRes              x86_64      4:1.9725-3.el7        base       45 k
 perl-Time-Local              noarch      1.2300-2.el7          base       24 k
 perl-constant                noarch      1.27-2.el7            base       19 k
 perl-libs                    x86_64      4:5.16.3-285.el7      base      687 k
 perl-macros                  x86_64      4:5.16.3-285.el7      base       42 k
 perl-parent                  noarch      1:0.225-244.el7       base       12 k
 perl-podlators               noarch      2.5.1-3.el7           base      112 k
 perl-threads                 x86_64      1.87-4.el7            base       49 k
 perl-threads-shared          x86_64      1.43-6.el7            base       39 k
 tncfhh                       x86_64      0.8.3-16.el7          base      680 k
 tncfhh-libs                  x86_64      0.8.3-16.el7          base      160 k
 tncfhh-utils                 x86_64      0.8.3-16.el7          base       33 k
 xerces-c                     x86_64      3.1.1-6.el7           base      878 k

Transaction Summary
================================================================================
Install  2 Packages (+44 Dependent packages)

Total download size: 16 M
Installed size: 51 M
Is this ok [y/d/N]:
```

### Configure FreeRADIUS

{% include important.html content="<br/>
This solution's use of FreeRADIUS must run as root to access the .google_authenticator in the user's home directory. Verified by changing user back to radiusd breaks authentication on May 21, 2015."
%}

Edit radiusd.conf

```
[root@2fcosrad7 ~]# vi /etc/raddb/radiusd.conf
```

Locate `user` and `group`

```
user = radiusd
group = radiusd
```

Update

```
#user = radiusd
#group = radiusd
user = root
group = root
```

Edit sites-enabled/default

```
[root@2fcosrad7 ~]# vi /etc/raddb/sites-enabled/default
```

Locate `pam`

```
        #  Pluggable Authentication Modules.
#       pam
```

Update

```
        #  Pluggable Authentication Modules.
        pam
```

Enable pam module

```
[root@2fcosrad7 ~]# ln -s /etc/raddb/mods-available/pam /etc/raddb/mods-enabled/pam
```

Results with

```
[root@2fcosrad7 ~]# ls /etc/raddb/mods-enabled/pam
/etc/raddb/mods-enabled/pam
```

Configure clients.conf

```
[root@2fcosrad7 ~]# vi /etc/raddb/clients.conf
```

Add the following above "client localhost {" where the IP address is the client or VPN solution, e.g. Juniper SSL or SoftEther. Don't use my example "secret123" but a shared secret with 12 to 16 upper and lower case characters, numbers, and symbols.

```
client 172.16.1.23 {
        ipaddr = 172.16.1.23
        secret = secret123
        require_message_authenticator = no
        nas_type = other
}
```

Configure 'users'

```
[root@2fcosrad7 ~]# vi /etc/raddb/users
```

Locate the following

```
#DEFAULT Group == "disabled", Auth-Type := Reject
# Reply-Message = "Your account has been disabled."
#
```

Update as follows

```
DEFAULT Group == "disabled", Auth-Type := Reject
Reply-Message = "Your account has been disabled."

DEFAULT Auth-Type := PAM
```

### Test FreeRADIUS with an UNIX account

Start radiusd in debug mode

```
[root@2fcosrad7 ~]# radiusd -X
```

Open a additional shell or SSH.

Create user

```
[root@2fcosrad7 ~]# useradd raduser
[root@2fcosrad7 ~]# passwd raduser
Changing password for user raduser.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
```

Use radtest from radiusd-util package using the local unix account, raduser.

```
[root@2fcosrad7 ~]# radtest raduser Password1 localhost 0 testing123
Sending Access-Request Id 194 from 0.0.0.0:39289 to 127.0.0.1:1812
        User-Name = 'raduser'
        User-Password = 'Password1'
        NAS-IP-Address = 172.16.1.25
        NAS-Port = 0
        Message-Authenticator = 0x00
Received Access-Accept Id 194 from 127.0.0.1:1812 to 127.0.0.1:39289 length 20
```

**Received Access-Accept** should be the response, otherwise you will receive a reject. If so, backup and check your work and correct errors before proceeding.

## SSSD

### Install SSSD

```
[root@2fcosrad7 ~]# yum install sssd realmd adcli
```

Results with

```
================================================================================
 Package                Arch        Version                  Repository    Size
================================================================================
Installing:
 adcli                  x86_64      0.7.5-4.el7              base          96 k
 realmd                 x86_64      0.14.6-6.el7             base         234 k
 sssd                   x86_64      1.12.2-58.el7_1.6        updates       79 k
Installing for dependencies:
 PackageKit-glib        x86_64      0.8.9-11.el7.centos      base         124 k
 bind-libs              x86_64      32:9.9.4-18.el7_1.1      updates      1.0 M
 bind-utils             x86_64      32:9.9.4-18.el7_1.1      updates      199 k
 c-ares                 x86_64      1.10.0-3.el7             base          78 k
 cups-libs              x86_64      1:1.6.3-17.el7           base         354 k
 cyrus-sasl-gssapi      x86_64      2.1.26-17.el7            base          40 k
 libarchive             x86_64      3.1.2-7.el7              base         317 k
 libbasicobjects        x86_64      0.1.1-24.el7             base          24 k
 libcollection          x86_64      0.6.2-24.el7             base          40 k
 libdhash               x86_64      0.4.3-24.el7             base          27 k
 libini_config          x86_64      1.1.0-24.el7             base          50 k
 libipa_hbac            x86_64      1.12.2-58.el7_1.6        updates       85 k
 libldb                 x86_64      1.1.17-2.el7             base         122 k
 libnfsidmap            x86_64      0.25-11.el7              base          46 k
 libpath_utils          x86_64      0.2.1-24.el7             base          27 k
 libref_array           x86_64      0.1.4-24.el7             base          26 k
 libsmbclient           x86_64      4.1.12-21.el7_1          base         120 k
 libsss_idmap           x86_64      1.12.2-58.el7_1.6        updates       90 k
 libsss_nss_idmap       x86_64      1.12.2-58.el7_1.6        updates       89 k
 libtdb                 x86_64      1.3.0-1.el7              base          44 k
 libtevent              x86_64      0.9.21-3.el7             base          31 k
 libwbclient            x86_64      4.1.12-21.el7_1          base          89 k
 oddjob                 x86_64      0.31.5-4.el7             base          69 k
 oddjob-mkhomedir       x86_64      0.31.5-4.el7             base          38 k
 psmisc                 x86_64      22.20-8.el7              base         140 k
 pytalloc               x86_64      2.1.1-1.el7              base          13 k
 python-sssdconfig      noarch      1.12.2-58.el7_1.6        updates      111 k
 samba-common           x86_64      4.1.12-21.el7_1          base         708 k
 samba-libs             x86_64      4.1.12-21.el7_1          base         4.3 M
 sssd-ad                x86_64      1.12.2-58.el7_1.6        updates      184 k
 sssd-client            x86_64      1.12.2-58.el7_1.6        updates      142 k
 sssd-common            x86_64      1.12.2-58.el7_1.6        updates      1.0 M
 sssd-common-pac        x86_64      1.12.2-58.el7_1.6        updates      118 k
 sssd-ipa               x86_64      1.12.2-58.el7_1.6        updates      220 k
 sssd-krb5              x86_64      1.12.2-58.el7_1.6        updates      115 k
 sssd-krb5-common       x86_64      1.12.2-58.el7_1.6        updates      174 k
 sssd-ldap              x86_64      1.12.2-58.el7_1.6        updates      195 k
 sssd-proxy             x86_64      1.12.2-58.el7_1.6        updates      112 k

Transaction Summary
================================================================================
Install  3 Packages (+38 Dependent packages)

Total download size: 11 M
Installed size: 31 M
Is this ok [y/d/N]:
```

Join host, 2fcosrad7, to the domain 2factor.net. You can also use `-U` to specify an administrator account. Use `man realm` to review options including specifying the OU to create the computer object versus the default "Computers" OU.

```
[root@2fcosrad7 ~]# realm join 2factor.net
Password for Administrator:
```

After you provide valid credentials with the appropriate privileges, a computer object and DDNS record are created.

It is a good idea to limit what users cans access using a Active Directory user credentials through the use of a Active Directory security group. I created a group called vpnusers and added richard@2factor.net as a memember.

Specify an existing Active Directory group, e.g. vpnusers.

```
realm permit -g vpnusers
```

Alternatively, you use --group versus -g

### Test SSSD

Utilize a group member, richard, of vpnusers to login at the console or SSH.

```
CentOS Linux 7 (Core)
Kernel 3.10.0-229.1.2.el7.x86_64 on an x86_64

2fcosrad7 login: richard@2factor.net
Password:
Creating home directory for richard@2factor.net.
[richard@2factor.net@2fcosrad7 ~]$
```

### Test FreeRADIUS with a SSSD account

Start FreeRADIUS in debug mode

```
[root@2fcosrad7 ~]# radiusd -X
```

Open another shell and use the `radtest` utility and use an Active Directory user that is a member of the group vpnusers.

```
[root@2fcosrad7 ~]# radtest richard@2factor.net Password1 localhost 0 testing123
```

Results should containt `Access-Accept` otherwise, backup and check your work.

```
Sending Access-Request Id 251 from 0.0.0.0:53104 to 127.0.0.1:1812
        User-Name = 'richard@2factor.net'
        User-Password = 'Password1'
        NAS-IP-Address = 172.16.1.25
        NAS-Port = 0
        Message-Authenticator = 0x00
Received Access-Accept Id 251 from 127.0.0.1:1812 to 127.0.0.1:53104 length 20
```

Success results with "Received Access-Accept." Use 'ctrl+c' to kill the radiusd daemon.

## Google Authenticator

### Install compile requirements

```
[root@2fcosrad7 ~]# yum install pam-devel make gcc-c++ git
```

Results with

```
================================================================================
 Package               Arch        Version                   Repository    Size
================================================================================
Installing:
 gcc-c++               x86_64      4.8.3-9.el7               base         7.2 M
 git                   x86_64      1.8.3.1-4.el7             base         4.3 M
 pam-devel             x86_64      1.1.8-12.el7              base         183 k
Installing for dependencies:
 cpp                   x86_64      4.8.3-9.el7               base         5.9 M
 gcc                   x86_64      4.8.3-9.el7               base          16 M
 glibc-devel           x86_64      2.17-78.el7               base         1.0 M
 glibc-headers         x86_64      2.17-78.el7               base         656 k
 kernel-headers        x86_64      3.10.0-229.4.2.el7        updates      2.3 M
 libgnome-keyring      x86_64      3.8.0-3.el7               base         109 k
 libmpc                x86_64      1.0.1-3.el7               base          51 k
 libstdc++-devel       x86_64      4.8.3-9.el7               base         1.5 M
 mpfr                  x86_64      3.1.1-4.el7               base         203 k
 perl-Error            noarch      1:0.17020-2.el7           base          32 k
 perl-Git              noarch      1.8.3.1-4.el7             base          52 k
 perl-TermReadKey      x86_64      2.30-20.el7               base          31 k
 rsync                 x86_64      3.0.9-15.el7              base         359 k

Transaction Summary
================================================================================
Install  3 Packages (+13 Dependent packages)

Total download size: 40 M
Installed size: 107 M
Is this ok [y/d/N]:
```

### Obtain Source

Note the current directory is root's home or `~`
```
[root@2fcosrad7 ~]# git clone https://code.google.com/p/google-authenticator/
```

Results with
```
Cloning into 'google-authenticator'...
remote: Counting objects: 1056, done.
Receiving objects: 100% (1056/1056), 2.27 MiB | 920.00 KiB/s, done.
Resolving deltas: 100% (509/509), done.
```

### Build Binaries

Build the libpam found in /google-authenticator/libpam.

```
[root@2fcosrad7 ~]# cd ~/google-authenticator/libpam/
[root@2fcosrad7 libpam]# make
```

Results with

```
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o google-authenticator.o google-authenticator.c
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o base32.o base32.c
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o hmac.o hmac.c
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o sha1.o sha1.c
gcc -g   -o google-authenticator google-authenticator.o base32.o hmac.o sha1.o  -ldl
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o pam_google_authenticator.o pam_google_authenticator.c
gcc -shared -g   -o pam_google_authenticator.so pam_google_authenticator.o base32.o hmac.o sha1.o -lpam
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o demo.o demo.c
demo.c: In function ‘pam_get_item’:
demo.c:88:36: warning: argument to ‘sizeof’ in ‘memcpy’ call is the same expression as the source; did you mean to remove the addressof? [-Wsizeof-pointer-memaccess]
       memcpy(item, &service, sizeof(&service));
                                    ^
demo.c:93:33: warning: argument to ‘sizeof’ in ‘memcpy’ call is the same expression as the source; did you mean to remove the addressof? [-Wsizeof-pointer-memaccess]
       memcpy(item, &user, sizeof(&user));
                                 ^
gcc -DDEMO --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o pam_google_authenticator_demo.o pam_google_authenticator.c
gcc -g   -rdynamic -o demo demo.o pam_google_authenticator_demo.o base32.o hmac.o sha1.o  -ldl
gcc -DTESTING --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden        \
              -o pam_google_authenticator_testing.o pam_google_authenticator.c
gcc -shared -g   -o pam_google_authenticator_testing.so pam_google_authenticator_testing.o base32.o hmac.o sha1.o -lpam
gcc --std=gnu99 -Wall -O2 -g -fPIC -c  -fvisibility=hidden  -o pam_google_authenticator_unittest.o pam_google_authenticator_unittest.c
pam_google_authenticator_unittest.c: In function ‘pam_get_item’:
pam_google_authenticator_unittest.c:79:36: warning: argument to ‘sizeof’ in ‘memcpy’ call is the same expression as the source; did you mean to remove the addressof? [-Wsizeof-pointer-memaccess]
       memcpy(item, &service, sizeof(&service));
                                    ^
pam_google_authenticator_unittest.c:84:33: warning: argument to ‘sizeof’ in ‘memcpy’ call is the same expression as the source; did you mean to remove the addressof? [-Wsizeof-pointer-memaccess]
       memcpy(item, &user, sizeof(&user));
                                 ^
gcc -g   -rdynamic -o pam_google_authenticator_unittest pam_google_authenticator_unittest.o base32.o hmac.o sha1.o -lc  -ldl
```

### Install binaries

```
[root@2fcosrad7 libpam]# make install
```

Results with:

```
cp pam_google_authenticator.so /lib64/security
cp google-authenticator /usr/local/bin
```

### Setup User

On-board the user using su and google-authenticator command.

```
[root@2fcosrad7 ~]# su - richard@2factor.net
Last login: Tue May 12 20:27:41 EDT 2015 from 172.16.1.1 on pts/1
[richard@2factor.net@2fcosrad7 ~]$ google-authenticator
```

Responding with `y` to queries results with

```
Do you want authentication tokens to be time-based (y/n) y
https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/richard@2factor.net@2fcosrad7.2factor.net%3Fsecret%3DFRL4H7J4OOCY4QGA

[QR image]

Your new secret key is: FRL4H7J4OOCY4QGA
Your verification code is 131105
Your emergency scratch codes are:
  86104574
  16057979
  31558510
  31258658
  14698995

Do you want me to update your "/home/2factor.net/richard/.google_authenticator" file (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, tokens are good for 30 seconds and in order to compensate for
possible time-skew between the client and the server, we allow an extra
token before and after the current time. If you experience problems with poor
time synchronization, you can increase the window from its default
size of 1:30min to about 4min. Do you want to do so (y/n) y

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting (y/n) y
```

The secret key is required to configure the Google Authenticator App so note and secure it in a safe place. The emergency codes are evil but great for testing so copy/paste for initial testing.

## PAM

The `/etc/pam.d/radiusd` file needs to be configured to utilize both SSSD and Google Authenticator.

Below is the default radiusd pam.d file

```
#%PAM-1.0
auth       include     password-auth
account    required    pam_nologin.so
account    include     password-auth
password   include     password-auth
session    include     password-auth
```

Edit and save the file

```
#%PAM-1.0
auth       requisite    pam_google_authenticator.so forward_pass
auth       required     pam_sss.so use_first_pass
account    required     pam_nologin.so
account    include      password-auth
session    include      password-auth
```

{% include note.html content="<br/>
Alternative configuration is to remove <b>pam_sss.so</b> and set <b>auth required pam_google_authenticator.so</b> (no forward_pass or use_first_pass) as required. This results in being prompted for the token only. When integrating with applications, you would chain authentication for different authentication sources. For example, a VPN solution could query Active Directory for user credentials, pass forward the user account and prompt for the token.<br/>
<br/>
Account lock-outs need to be taken into consideration! For example, if querying AD first, if you do not take precautions to set thresholds lower than AD, the user account could be locked. Results with the user unable to use their AD credentials to access any services utilizing AD credential during the lock out period, e.g. fifteen minutes."
%}

## SELinux = permissive

By default SELinux is set to "enforcing" which will prevent access for the FreeRadius service, radiusd, accessing user directories. However, radiusd needs access to `~/.google_authenticator` and as such we need to change from "enforcing" to "permissive."

View current SELinux mode using `getenforce`

```
[root@2fcosrad7 ~]# getenforce
Enforcing
```

or alternatively using `sestatus`

```
[root@2fcosrad7 ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28
```

To set SELinux to "permissive" execute the command below but it will revert to enforcing next reboot.

```
[root@2fcosrad7 ~]# setenforce permissive
[root@2fcosrad7 ~]# getenforce
Permissive
```

Edit /etc/selinux/config to set SELinux's mode to permissive which will be persistent across system reboot.

```
[root@2fcosrad7 ~]# vi /etc/selinix/config
```

Update SELINUX = enforcing to permissive

```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

## Test FreeRADIUS with SSSD & Google Authenticator

Using the radtest utility, enter the user `richard@2factor.net` which is a member of vpnusers followed with both.

```
radtest <username> (<active directory pasword><google-authenticator emergency code>) localhost 0 testing123
```

In my case

```
radtest richard@2factor.net Password186104574 localhost 0 testing123
```

Results with

```
[root@2fcosrad7 ~]# radtest richard@2factor.net Password186104574 localhost 0 testing123
Sending Access-Request Id 134 from 0.0.0.0:38338 to 127.0.0.1:1812
        User-Name = 'richard@2factor.net'
        User-Password = 'Password186104574'
        NAS-IP-Address = 172.16.1.25
        NAS-Port = 0
        Message-Authenticator = 0x00
Received Access-Accept Id 134 from 127.0.0.1:1812 to 127.0.0.1:38338 length 20
```

Use 'ctrl+c' to kill the radiusd process.

## Firewall

The assumption is you have not disabled or stopped the firewall. If you have, execute the following two command.

```
[root@2fcosrad7 ~]# systemctl enable firewalld
[root@2fcosrad7 ~]# systemctl start firewalld
```

Determine default zone.

```
[root@2fcosrad7 ~]# firewall-cmd --get-default-zone
public
```

List current rules for public zone.

```
[root@2fcosrad7 ~]# firewall-cmd --zone=public --list-all
public (default, active)
  interfaces: eno16777728
  sources:
  services: dhcpv6-client ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

List available services. Note "radius."

```
[root@2fcosrad7 ~]# firewall-cmd --get-services | grep rad
RH-Satellite-6 amanda-client bacula bacula-client dhcp dhcpv6 dhcpv6-client dns ftp high-availability http https imaps ipp ipp-client ipsec kerberos kpasswd ldap ldaps libvirt libvirt-tls mdns mountd ms-wbt mysql nfs ntp openvpn pmcd pmproxy pmwebapi pmwebapis pop3s postgresql proxy-dhcp radius rpc-bind samba samba-client smtp ssh telnet tftp tftp-client transmission-client vnc-server wbem-https
```

Permit Radius ports and protocols for public zone.

```
[root@2fcosrad7 ~]# firewall-cmd --permanent --zone=public --add-service=radius
success
```

Reload rules otherwise the above command will not take effect without a reboot.

```
[root@2fcosrad7 ~]# firewall-cmd --reload
```

List rules for the default zone, public. Note the addition of "radius" for `--list-service` or, alternatively, `--list-all`.

```
[root@2fcosrad7 ~]# firewall-cmd --list-service --zone=public
dhcpv6-client radius ssh
[root@2fcosrad7 ~]# firewall-cmd --zone=public --list-all
public (default, active)
  interfaces: eno16777728
  sources:
  services: dhcpv6-client radius ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

References:

https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Security_Guide/sec-Using_Firewalls.html

### Radiusd service

Need to enable and start the radiusd service. You killed your prior instances of radiusd using ctrl+c for testing, right?

```
[root@2fcosrad7 ~]# systemctl enable radiusd
ln -s '/usr/lib/systemd/system/radiusd.service' '/etc/systemd/system/multi-user.target.wants/radiusd.service'
[root@2fcosrad7 ~]# systemctl start radiusd
```

## Install Google Authenticator App

Obtain Google Authenticator App for your mobile device via Google Play Store and setup using your secret key, e.g. `FRL4H7J4OOCY4QGA`.

## Test FreeRADIUS with NAS

For the purpose of this tutorial, my Network Access Server (NAS) is PulseSecure SSL VPN solution, formerly known as Juniper, and its IP address is 172.16.1.23. Above we created the connection under the section titled "FreeRADIUS configuration" using configuration file clients.conf.

After configuration of the NAS--I will not illustrate the configuration, but, obviously, use <i>radius</i>, connect to the login portal.

### Log-in process

1. For user, enter `richard@2factor.net`
1. For password, enter `[<active_directory_password>+<google_authenticator_otp_token>]` or `Password1437247`

If all goes well, you should successfully login.

## Done?!

At this point, we have completed the basic build. The next section "Tidy Up!" provides a number of additional suggestions, but are not requirements.

## Tidy Up!

### Delete local test user, raduser

```
[root@2fcosrad7 ~]# userdel -r raduser
```

### Update clients.conf

Update the /etc/raddb/clients.conf file with an appropriate password for all secrets including the `localhost` connection. For example, change the default localhost from "testing123" to a secret with 12 to 16 upper and lower case characters, numbers, and symbols.

```
[root@2fcosrad7 ~]# vi /etc/raddb/clients.conf
```

### Move FreeRADIUS Computer Object

Ask your Active Directory administrator to move the computer object for your FreeRADIUS host from "Computers" to the correct Active Directory Organization Unit (OU). It's probably not "Computers" and share it is a Linux host for she or he may create a new OU for Linux hosts.

### Enable FreeRADIUS Logs

By default, logs are not enabled.

Update the sites-enabled/default to log accounting information.

```
[root@2fcosrad7 ~]# vi /etc/raddb/sites-enabled/default
```

From the authorize section in /etc/raddb/sites-enabled/default file:

```
[root@2fcosrad7 ~]# vi /etc/raddb/sites-enabled/default

        #
        #  If you want to have a log of authentication requests,
        #  un-comment the following line, and the 'detail auth_log'
        #  section, above.
#       auth_log
```

Update to

```
        auth_log
```


From the post-auth section in /etc/raddb/sites-enabled/default file:

```
        #
        #  If you want to have a log of authentication replies,
        #  un-comment the following line, and the 'detail reply_log'
        #  section, above.
#       reply_log
```

Update to

```
        reply_log
```

There is a dependency on the /etc/raddb/mods-enabled/detail.log, however, by default no changes need to be made. After auth_log and reply_log changes are made, it just works.

Restart FreeRADIUS

```
[root@2fcosrad7 ~]# systemctl restart radiusd
```

Login via you NAS, SSL VPN, and verify logs are generated.

```
[root@2fcosrad7 ~]# ls /var/log/radius/radacct/
172.16.1.23
[root@2fcosrad7 ~]# ls /var/log/radius/radacct/172.16.1.23/
auth-detail-20150520  reply-detail-20150520
[root@2fcosrad7 ~]# cat /var/log/radius/radacct/172.16.1.23/auth-detail-20150520
Wed May 20 17:01:13 2015
        Packet-Type = Access-Request
        NAS-Identifier = '172.16.1.23'
        User-Name = 'richard@2factor.net'
        Tunnel-Client-Endpoint:0 = '172.16.1.1'
        NAS-IP-Address = 172.16.1.23
        NAS-Port = 0
        Acct-Session-Id = 'richard@2factor.net(2frad user realm)\"Wed May 20 17:01:13 2015\"/l0aCMuI'
        Event-Timestamp = 'May 20 2015 17:01:13 PDT'
```

### SELinux Access Vector Cache (AVC) Denies

The process below will successfully create a module to resolve radiusd "avc" errors in `/var/log/audit/audit.log`, however, SELinux will continue to prevent access to .google_authenticator file--I am still working to resolve (May, 22, 2015). As such, SELinux will continue to be in permissive mode, but we can remove 'avc' errors and benefit when doing analysis of the audit.log.

Install SELinux utilities

```
[root@2fcosrad7 ~]# yum install policycoreutils-python
```

Results with

```
================================================================================
 Package                      Arch         Version             Repository  Size
================================================================================
Installing:
 policycoreutils-python       x86_64       2.2.5-15.el7        base       434 k
Installing for dependencies:
 audit-libs-python            x86_64       2.4.1-5.el7         base        69 k
 checkpolicy                  x86_64       2.1.12-6.el7        base       247 k
 libcgroup                    x86_64       0.41-8.el7          base        64 k
 libsemanage-python           x86_64       2.1.10-16.el7       base        94 k
 python-IPy                   noarch       0.75-6.el7          base        32 k
 setools-libs                 x86_64       3.3.7-46.el7        base       485 k

Transaction Summary
================================================================================
Install  1 Package (+6 Dependent packages)

Total download size: 1.4 M
Installed size: 4.5 M
Is this ok [y/d/N]:
```

With our previous testing and SELinux set to "permissive," we have audit entries in audit.log. If you disabled SELinux, you will need to set to permissive, login using Google Authentication via radiusd, then follow the example below.

```
[root@2fcosrad7 ~]# audit2allow -a -M radiusd-gauth
```

Alternatively, you can use grep to filter for specific events

```
[root@2fcosrad7 ~]# grep radiusd /var/log/audit/audit.log | audit2allow -a -M radiusd-gauth
```

Results with

```
******************** IMPORTANT ***********************
To make this policy package active, execute:

[root@2fcosrad7 ~]# semodule -i radiusd-gauth.pp
```

Activate the policy package

```
[root@2fcosrad7 ~]# semodule -i radiusd-gauth.pp
```

That is it. You should receive no further avc errors.

{% include note.html content="<br/>
Disabling or enabling a module use <b>-d</b> or <b>-e</b>, respectively.<br/>
<b># semanage module -d radiusd-gauth</b><br/>
<br/>
Removing a module use <b>-r</b> to remove a module<br/>
<b># semanage module -r radiusd-gauth</b>"
%}

### Packet Capture

Once I have a working solution, I feel it is invaluable to capture a user login via tcpdump for comparison when things break in the future. This is how I do it.

Display available adapters

```
[root@2fcosrad7 ~]# tcpdump -D
1.nflog (Linux netfilter log (NFLOG) interface)
2.nfqueue (Linux netfilter queue (NFQUEUE) interface)
3.eno16777728
4.any (Pseudo-device that captures on all interfaces)
5.lo
```

Capture the transactions and write to file.

```
[root@2fcosrad7 ~]# tcpdump -i eno16777728 -w ~/radiusd.cap
tcpdump: listening on eno16777728, link-type EN10MB (Ethernet), capture size 65535 bytes
^C279 packets captured
280 packets received by filter
0 packets dropped by kernel
```

*Note* the ^C above which is `ctrl+c` to stop the tcpdump process.

Now using scp or winscp, copy the capture to a workstation for analysis using Wireshark or you favorite packet analysis tool then archive it for future use.

{% include links.html %}
