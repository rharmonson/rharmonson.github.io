---
title: SoftEther 4 Installation Guide on CentOS 7
tags: [linux centos sysadmin security]
last_updated: September 14, 2016
keywords: linux centos system administration remote access vpn softether
summary: "The purpose of this article is to describe how to SoftEther VPN Server and Client on CentOS 7."
layout: default_toc
sidebar:
toc: false
permalink: cos7se4inst.html
folder: mydoc
---

# SoftEther 4 Installation Guide on CentOS 7

This article provides instructions to implement SoftEther VPN Server and Client on CentOS 7.2.1511 and Fedora 24, respectively. SoftEther VPN is an open source VPN solution that can be used for secure client remote access VPN or branch offices site to site VPN. It supports most portable devices and operating systems as well as emulating several other VPN standards such as Cisco IPSEC and OpenVPN.

This article's focus will be secure client remote access using SoftEther VPN's native protocol.

## SoftEther VPN Server

### CentOS 7 Install
Begin by completing a base operating system build to your preference or you can follow my guide found here:

https://rharmonson.github.io/cos7inst.html


### Install Requirements
After the base OS installation and configuration, install SoftEther VPN Server's build dependencies.

**Requirements**

* gcc software
* binutils software
* tar, gzip or other software for extracting package files
* chkconfig system utility
* cat, cp or other basic file operation utility
* EUC-JP, UTF-8 or other code page table for use in a Japanese language environment
* libc (glibc) library
* zlib library
* openssl library
* readline library
* ncurses library
* pthread library

To meet the requirements above, execute the command below to the install the packages and their dependencies.

```
# yum install gcc zlib-devel openssl-devel readline-devel ncurses-devel
```

Results
```
Dependencies Resolved

================================================================================
 Package                 Arch       Version                   Repository   Size
================================================================================
Installing:
 gcc                     x86_64     4.8.5-4.el7               base         16 M
 ncurses-devel           x86_64     5.9-13.20130511.el7       base        713 k
 openssl-devel           x86_64     1:1.0.1e-51.el7_2.5       updates     1.2 M
 readline-devel          x86_64     6.2-9.el7                 base        138 k
 zlib-devel              x86_64     1.2.7-15.el7              base         50 k
Installing for dependencies:
 cpp                     x86_64     4.8.5-4.el7               base        5.9 M
 glibc-devel             x86_64     2.17-106.el7_2.8          updates     1.0 M
 glibc-headers           x86_64     2.17-106.el7_2.8          updates     663 k
 kernel-headers          x86_64     3.10.0-327.28.2.el7       updates     3.2 M
 keyutils-libs-devel     x86_64     1.5.8-3.el7               base         37 k
 krb5-devel              x86_64     1.13.2-12.el7_2           updates     649 k
 libcom_err-devel        x86_64     1.42.9-7.el7              base         30 k
 libmpc                  x86_64     1.0.1-3.el7               base         51 k
 libselinux-devel        x86_64     2.2.2-6.el7               base        174 k
 libsepol-devel          x86_64     2.1.9-3.el7               base         71 k
 libverto-devel          x86_64     0.2.5-4.el7               base         12 k
 mpfr                    x86_64     3.1.1-4.el7               base        203 k
 pcre-devel              x86_64     8.32-15.el7_2.1           updates     479 k

Transaction Summary
================================================================================
Install  5 Packages (+13 Dependent packages)

Total download size: 31 M
Installed size: 67 M
Is this ok [y/d/N]:
```

### SELinux

Prior to installation and testing, I updated `/etc/selinux/config` from `enforcing` to `permissive`. After testing, I set back to `enforcing`. No problem so far. *knock-on-wood*. It may have been an unnecessary step.

### Firewall

If you followed my CentOS 7 1511 Minimal guide, firewalld was ripped out and the iptables-services package was installed. If not, revise the script below as necessary. The primary change to my default iptables filter is the addition of `iptables -I INPUT -p tcp --dport 443 -j ACCEPT`. This assumes you will be using the SoftEther VPN client on port 443. After installation and testing, creating an alternative port is painless.

Create file, `vi softether.fw`

```
#!/bin/bash
# Set default chain policies
iptables -P INPUT DROP
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Accept on localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established sessions to receive traffic
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

#Permit ICMP Echo (OPTIONAL)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Accept incoming SSH
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

# Accept incoming HTTPS for SoftEther (default)
iptables -I INPUT -p tcp --dport 443 -j ACCEPT

# Save Changes
service iptables save

# Service
systemctl restart iptables
systemctl status iptables
```

Set the file to executable using `chmod +x softether.fw` then execute `./softether.fw`. Review the change using `iptables -L -n -v`.

### Install SoftEther VPN Server

#### Download

Using a browser, go to `http://www.softether-download.com/files/softether/` and locate the version of the product to install. Copy or type the link location as follows to download using curl.

```
# curl -O http://www.softether-download.com/files/softether/v4.20-9608-rtm-2016.04.17-tree/Linux/SoftEther_VPN_Server/64bit_-_Intel_x64_or_AMD64/softether-vpnserver-v4.20-9608-rtm-2016.04.17-linux-x64-64bit.tar.gz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 6117k  100 6117k    0     0   451k      0  0:00:13  0:00:13 --:--:--  726k
```

#### Unpack

Unpack the archive using tar.

```
[root@sevpn ~]# tar xzvf softether-vpnserver-v4.20-9608-rtm-2016.04.17-linux-x64-64bit.tar.gz -C /usr/local/
vpnserver/
vpnserver/Makefile
vpnserver/.install.sh
vpnserver/ReadMeFirst_License.txt
vpnserver/Authors.txt
vpnserver/ReadMeFirst_Important_Notices_ja.txt
vpnserver/ReadMeFirst_Important_Notices_en.txt
vpnserver/ReadMeFirst_Important_Notices_cn.txt
vpnserver/code/
vpnserver/code/vpnserver.a
vpnserver/code/vpncmd.a
vpnserver/lib/
vpnserver/lib/libcharset.a
vpnserver/lib/libcrypto.a
vpnserver/lib/libedit.a
vpnserver/lib/libiconv.a
vpnserver/lib/libintelaes.a
vpnserver/lib/libncurses.a
vpnserver/lib/libssl.a
vpnserver/lib/libz.a
vpnserver/lib/License.txt
vpnserver/hamcore.se2
```

#### Compile

Time to compile or make SoftEther.

{% include warning.html content="grep command not found<br/>
A number of readers have reported the error which indicates a missing dependency, command which, to compile SoftEther with make.<br/>
<br/>
Solution: yum install which.<br/>
<br/>
Readers dbaflight, shaunie2fly, danuid, and <i>cguroo</i> reported the problem. Thank you!"
%}

```
# cd /usr/local/vpnserver
# make
```

Results

```
make[1]: Entering directory `/usr/local/vpnserver'
Preparing SoftEther VPN Server...
ranlib lib/libcharset.a
ranlib lib/libcrypto.a
ranlib lib/libedit.a
ranlib lib/libiconv.a
ranlib lib/libintelaes.a
ranlib lib/libncurses.a
ranlib lib/libssl.a
ranlib lib/libz.a
ranlib code/vpnserver.a
gcc code/vpnserver.a -O2 -fsigned-char -pthread -m64 -lm -ldl -lrt -lpthread -L./ lib/libssl.a lib/libcrypto.a lib/libiconv.a lib/libcharset.a lib/libedit.a lib/libncurses.a lib/libz.a lib/libintelaes.a -o vpnserver
ranlib code/vpncmd.a
gcc code/vpncmd.a -O2 -fsigned-char -pthread -m64 -lm -ldl -lrt -lpthread -L./ lib/libssl.a lib/libcrypto.a lib/libiconv.a lib/libcharset.a lib/libedit.a lib/libncurses.a lib/libz.a lib/libintelaes.a -o vpncmd
./vpncmd /tool /cmd:Check
vpncmd command - SoftEther VPN Command Line Management Utility
SoftEther VPN Command Line Management Utility (vpncmd command)
Version 4.20 Build 9608   (English)
Compiled 2016/04/17 21:59:35 by yagi at pc30
Copyright (c) SoftEther VPN Project. All Rights Reserved.

VPN Tools has been launched. By inputting HELP, you can view a list of the commands that can be used.

VPN Tools>Check
Check command - Check whether SoftEther VPN Operation is Possible
---------------------------------------------------
SoftEther VPN Operation Environment Check Tool

Copyright (c) SoftEther VPN Project.
All Rights Reserved.

If this operation environment check tool is run on a system and that system passes, it is most likely that SoftEther VPN software can operate on that system. This check may take a while. Please wait...

Checking 'Kernel System'...
              Pass
Checking 'Memory Operation System'...
              Pass
Checking 'ANSI / Unicode string processing system'...
              Pass
Checking 'File system'...
              Pass
Checking 'Thread processing system'...
              Pass
Checking 'Network system'...
              Pass

All checks passed. It is most likely that SoftEther VPN Server / Bridge can operate normally on this system.

The command completed successfully.


--------------------------------------------------------------------
The preparation of SoftEther VPN Server is completed !


*** How to switch the display language of the SoftEther VPN Server Service ***
SoftEther VPN Server supports the following languages:
  - Japanese
  - English
  - Simplified Chinese

You can choose your prefered language of SoftEther VPN Server at any time.
To switch the current language, open and edit the 'lang.config' file.


*** How to start the SoftEther VPN Server Service ***

Please execute './vpnserver start' to run the SoftEther VPN Server Background Service.
And please execute './vpncmd' to run the SoftEther VPN Command-Line Utility to configure SoftEther VPN Server.
Of course, you can use the VPN Server Manager GUI Application for Windows on the other Windows PC in order to configure the SoftEther VPN Server remotely.
--------------------------------------------------------------------

make[1]: Leaving directory `/usr/local/vpnserver'
```

### Start SoftEther

Verify vpnserver operations before continuing by starting SoftEther from command-line.
```
# cd /usr/local/vpnserver
# ./vpnserver start
```

To close the vpnserver, execute `./vpnserver close`.

### Permissions

Update file permissions to something a bit more sane.

```
[root@sevpn ~]
# chown -R root:root /usr/local/vpnserver  
# cd /usr/local/vpnserver/  
# chmod -R 600 *  
# chmod 700 vpncmd  
# chmod 700 vpnserver
```

### Create systemd Script

Create a systemd script to auto-start/stop SoftEther. Kudos to hsaito!

Reference: https://github.com/hsaito/SoftEtherVPN/blob/a9b9afc806a5df8598fd9acda2424d9c48ac8462/systemd/softether-vpnserver.service

```
# vi /etc/systemd/system/softether.service
```

```
[Unit]
Description=SoftEther VPN Server  
After=network.target auditd.service  
ConditionPathExists=!/usr/local/vpnserver/do_not_run

[Service]
Type=forking  
EnvironmentFile=-/usr/local/vpnserver  
ExecStart=/usr/local/vpnserver/vpnserver start  
ExecStop=/usr/local/vpnserver/vpnserver stop  
KillMode=process  
Restart=on-failure

# Hardening
PrivateTmp=yes  
ProtectHome=yes  
ProtectSystem=full  
ReadOnlyDirectories=/  
ReadWriteDirectories=-/usr/local/vpnserver  
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE CAP_NET_BROADCAST CAP_NET_RAW CAP_SYS_NICE CAP_SYS_ADMIN CAP_SETUID

[Install]
WantedBy=multi-user.target
```

Also, enable and start the service.

```
# systemctl enable vpnserver
# systemctl start vpnserver
```

### Configure SoftEther VPN Server

Next step is to use the vpncmd command or the SoftEther VPN Server Manager for Windows to configure SoftEther. The URL https://www.softether.org/4-docs/2-howto shows the various network scenarios or topographies for use with SoftEther. Our use case is "remote access," so an excellent starting point is ["Remote Access VPN to LAN"](https://www.softether.org/4-docs/2-howto/1.VPN_for_On-premise/2.Remote_Access_VPN_to_LAN) and ["Build a PC to LAN Remote Access VPN"](https://www.softether.org/4-docs/1-manual/A._Examples_of_Building_VPN_Networks/10.4_Build_a_PC-to-LAN_Remote_Access_VPN).

Next, I would advise following the instructions on configuring ["SecureNAT"](https://www.softether.org/index.php?title=4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.7_Virtual_NAT_%26_Virtual_DHCP_Servers). In my experience, it is the easiest to setup. Once the setup is complete, move to the section titled ["SoftEther VPN Client"](https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Secure-Remote-Access-with-SoftEther-VPN-Server-and-Client-4.2#softether-vpn-client).

### Local Bridge & dnsmasq (optional)

After the completion of your secure remote access service, use it. If the performance is acceptable, then move on to another project. However, if you feel that SecureNAT is sluggish, you may benefit using a local bridge. Begin by reading ["Measuring Effective Throughput"](https://www.softether.org/4-docs/1-manual/4._SoftEther_VPN_Client_Manual/4.8_Measuring_Effective_Throughput) and obtain metrics of your SecureNAT implementation. Then read ["Local Bridges"](https://www.softether.org/4-docs/1-manual/3._SoftEther_VPN_Server_Manual/3.6_Local_Bridges). Make sure you have disabled SecureNAT before implementing the local bridge. Obtain metrics using the local bridge and choose the solution that best meets your requirements.

#### systemd

If using a local bridge, update your systemd script to include an address assignment for the tap device.

```
#/etc/systemd/system/vpnserver.service
# chown root / chmod 644
[Unit]
Description=SoftEther VPN Server with TAP
After=network.target auditd.service  
ConditionPathExists=!/usr/local/vpnserver/do_not_run

[Service]
Type=forking  
EnvironmentFile=-/usr/local/vpnserver  
ExecStart=/usr/local/vpnserver/vpnserver start  
ExecStartPost=/bin/sleep 1  
ExecStartPost=/sbin/ip address add 192.168.10.254/24 dev tap_vpn  
ExecStop=/usr/local/vpnserver/vpnserver stop
KillMode=process  
Restart=on-failure

# Hardening
PrivateTmp=yes  
ProtectHome=yes  
ProtectSystem=full  
ReadOnlyDirectories=/  
ReadWriteDirectories=-/usr/local/vpnserver  
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE #CAP_NET_BROADCAST CAP_NET_RAW CAP_SYS_NICE CAP_SYS_ADMIN CAP_SETUID

[Install]
WantedBy=multi-user.target
```

#### dnsmasq

In addition, you may need DHCP. With SecureNAT, DHCP was provided but this is not the case (to my knowledge) using SoftEther VPN Server and a local bridge. Use dnsmasq.

Edit using `vi /etc/dnsmasq.conf` then copy and paste the following to the bottom of the file and revise for your environment.

```
# VPN Server Interface
interface=tap_vpn

# VPN Client DHCP Pool
dhcp-range=tap_vpn,192.168.10.10,192.168.10.200,255.255.255.0,4h

# Gateway
dhcp-option=tap_vpn,3,192.168.10.254

# DNS
dhcp-option=tap_vpn,6,192.168.1.1

# Domain
dhcp-option=tap_vpn,15,mydomain.net

# NTP
dhcp-option=tap_vpn,42,192.168.1.1
```

Enable and start dnsmasq using `systemctl enable dnsmasq` then `systemctl start dnsmasq`.

#### iptables

We need to permit DHCP for the bridge interface and configure a NAT. You can execute the changes from the shell or `iptables -f` (flush) the current policies and execute the updated script. The default INPUT policy is DROP, so use the console versus SSH.

```
#!/bin/bash

# Set default chain policies
iptables -P INPUT DROP
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Accept on localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established sessions to receive traffic
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

#Permit ICMP Echo (OPTIONAL)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Accept incoming SSH
iptables -I INPUT -p tcp --dport 22 -j ACCEPT

# SoftEther
iptables -I INPUT -p udp --dport 443 -j ACCEPT
iptables -I INPUT -p tcp --dport 443 -j ACCEPT

# DHCP (dnsmasq)
iptables -A INPUT -i tap_vpn -p udp --dport 67 -j ACCEPT

# NAT using Local Bridge
# 192.168.10.0/24 = Local Bridge & SoftEther VPN Clients (dnsmasq)
# 192.168.2.1 = SoftEther VPN Server's network interface
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -j SNAT --to-source 192.168.2.1

# Save Changes
service iptables save

# Service
systemctl restart iptables
systemctl status iptables
```

## SoftEther VPN Client

The Windows client works very well and requires little configuration outside of providing the correct host name or IP address, port, and credentials. It's interface, route, and DNS configuration is painless. The Linux client is not as seamless nor does the SoftEther VPN Project provide instruction. This section describes my method to setup the client on Fedora 24.

### Use Case

The use case is as follows:

* Fedora 24 Workstation
* Mobile user
* User has SUDO privileges
* VPN on demand versus always-on
* No split-tunnel access
* VPN Server with DHCP

Prior to proceeding, complete a Fedora 24 Workstation installation. I successfully tested Gnome, KDE, and several other Spins.

### Download Client

Obtain the SoftEther VPN Client from http://www.softether-download.com/files/softether/
using curl.

```
[john@wss ~]$ pwd
/home/john
[john@wss ~]$ mkdir temp
[john@wss ~]$ cd temp
[john@wss temp]$ curl -O http://www.softether-download.com/files/softether/v4.20-9608-rtm-2016.04.17-tree/Linux/SoftEther_VPN_Client/64bit_-_Intel_x64_or_AMD64/softether-vpnclient-v4.20-9608-rtm-2016.04.17-linux-x64-64bit.tar.gz

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 6116k  100 6116k    0     0   855k      0  0:00:07  0:00:07 --:--:-- 1248k
[john@wss temp]$

```

### Unpack & Compile

The compile requirements appear to be the same as the SoftEther VPN Server. With a default Fedora 24 Workstation installation, no additional packages were required to compile.

Using `tar`, unpack the file.

```
[john@wss temp]$ tar xzvf softether-vpnclient-v4.20-9608-rtm-2016.04.17-linux-x64-64bit.tar.gz
vpnclient/
vpnclient/Makefile
vpnclient/.install.sh
vpnclient/ReadMeFirst_License.txt
vpnclient/Authors.txt
vpnclient/ReadMeFirst_Important_Notices_ja.txt
vpnclient/ReadMeFirst_Important_Notices_en.txt
vpnclient/ReadMeFirst_Important_Notices_cn.txt
vpnclient/code/
vpnclient/code/vpnclient.a
vpnclient/code/vpncmd.a
vpnclient/lib/
vpnclient/lib/libcharset.a
vpnclient/lib/libcrypto.a
vpnclient/lib/libedit.a
vpnclient/lib/libiconv.a
vpnclient/lib/libintelaes.a
vpnclient/lib/libncurses.a
vpnclient/lib/libssl.a
vpnclient/lib/libz.a
vpnclient/lib/License.txt
vpnclient/hamcore.se2
```

`cd` into the vpnclient directory from the unpacked archive then compile using make.

```
[john@wss temp]$ cd vpnclient/
[john@wss vpnclient]$ make
--------------------------------------------------------------------

SoftEther VPN Client (Ver 4.20, Build 9608, Intel x64 / AMD64) for Linux Install Utility
Copyright (c) SoftEther Project at University of Tsukuba, Japan. All Rights Reserved.

--------------------------------------------------------------------


Do you want to read the License Agreement for this software ?

 1. Yes
 2. No

Please choose one of above number: 1

..skip

Did you read and understand the License Agreement ?
(If you couldn't read above text, Please read 'ReadMeFirst_License.txt'
 file with any text editor.)

 1. Yes
 2. No

Please choose one of above number:
1


Did you agree the License Agreement ?

1. Agree
2. Do Not Agree

Please choose one of above number:
1

make[1]: Entering directory '/home/john/temp/vpnclient'
Preparing SoftEther VPN Client...
ranlib lib/libcharset.a
ranlib lib/libcrypto.a
ranlib lib/libedit.a
ranlib lib/libiconv.a
ranlib lib/libintelaes.a
ranlib lib/libncurses.a
ranlib lib/libssl.a
ranlib lib/libz.a
ranlib code/vpnclient.a
gcc code/vpnclient.a -O2 -fsigned-char -pthread -m64 -lm -ldl -lrt -lpthread -L./ lib/libssl.a lib/libcrypto.a lib/libiconv.a lib/libcharset.a lib/libedit.a lib/libncurses.a lib/libz.a lib/libintelaes.a -o vpnclient
ranlib code/vpncmd.a
gcc code/vpncmd.a -O2 -fsigned-char -pthread -m64 -lm -ldl -lrt -lpthread -L./ lib/libssl.a lib/libcrypto.a lib/libiconv.a lib/libcharset.a lib/libedit.a lib/libncurses.a lib/libz.a lib/libintelaes.a -o vpncmd

--------------------------------------------------------------------
The preparation of SoftEther VPN Client is completed !


*** How to switch the display language of the SoftEther VPN Client Service ***
SoftEther VPN Client supports the following languages:
  - Japanese
  - English
  - Simplified Chinese

You can choose your preferred language of SoftEther VPN Client at any time.
To switch the current language, open and edit the 'lang.config' file.


*** How to start the SoftEther VPN Client Service ***

Please execute './vpnclient start' to run the SoftEther VPN Client Background Service.
And please execute './vpncmd' to run the SoftEther VPN Command-Line Utility to configure SoftEther VPN Client.
Of course, you can use the VPN Server Manager GUI Application for Windows on the other Windows PC in order to configure the SoftEther VPN Client remotely.
--------------------------------------------------------------------

make[1]: Leaving directory '/home/john/temp/vpnclient'
```

Change ownership to root, set permissions, then move /usr/local.

```
[john@wss vpnclient]$ chmod -R 600 *  
[john@wss vpnclient]$ chmod 700 vpncmd  
[john@wss vpnclient]$ chmod 700 vpnclient
[john@wss vpnclient]$ cd ..
[john@wss temp]$ sudo chown -R root.root vpnclient/
[john@wss temp]$ sudo mv vpnclient/ /usr/local/
```

### Start & Configure Client

The SoftEther VPN Client is started and stopped using `/usr/local/vpnclient/vpnclient start` or `/vpnclient stop`. To configure the client, use `vpncmd` as shown below. Substitute the values below as desired. Use `help` to show all available commands.

```
[john@wss temp]$ sudo /usr/local/vpnclient/vpnclient start
The SoftEther VPN Client service has been started.
[john@wss temp]$ sudo /usr/local/vpnclient/vpncmd
vpncmd command - SoftEther VPN Command Line Management Utility
SoftEther VPN Command Line Management Utility (vpncmd command)
Version 4.20 Build 9608   (English)
Compiled 2016/04/17 21:59:35 by yagi at pc30
Copyright (c) SoftEther VPN Project. All Rights Reserved.

By using vpncmd program, the following can be achieved.

1. Management of VPN Server or VPN Bridge
2. Management of VPN Client
3. Use of VPN Tools (certificate creation and Network Traffic Speed Test Tool)

Select 1, 2 or 3: 2

Specify the host name or IP address of the computer that the destination VPN Client is operating on.
If nothing is input and Enter is pressed, connection will be made to localhost (this computer).
Hostname of IP Address of Destination:

Connected to VPN Client "localhost".

VPN Client>niccreate
NicCreate command - Create New Virtual Network Adapter
Virtual Network Adapter Name: sev0

The command completed successfully.

VPN Client>accountcreate
AccountCreate command - Create New VPN Connection Setting
Name of VPN Connection Setting: myLab

Destination VPN Server Host Name and Port Number: ddns.domain.net:443


Destination Virtual Hub Name: vpn

Connecting User Name: john

Used Virtual Network Adapter Name: sev0

The command completed successfully.

VPN Client>accountpasswordset
AccountPasswordSet command - Set User Authentication Type of VPN Connection Setting to Password Authentication
Name of VPN Connection Setting: myLab

Please enter the password. To cancel press the Ctrl+D key.

Password: ********
Confirm input: ********


Specify standard or radius: standard

The command completed successfully.

VPN Client>accountstartupset
AccountStartupSet command - Set VPN Connection Setting as Startup Connection
Name of VPN Connection Setting: myLab

The command completed successfully.

VPN Client>exit
```

Stop SoftEther VPN Client.

```
[john@wss temp]$ sudo /usr/local/vpnclient/vpnclient stop
Stopping the SoftEther VPN Client service ...
SoftEther VPN Client service has been stopped.
[john@wss temp]$
```

Now we have a connection profile, user account and password, and the profile set to connect on start. Note that the vpncmd option `AccountStartupSet` sets the default profile, but you can use `AccountConnect` and `AccountDisconnect` to utilize different profiles.


{% include note.html content="You have two additional options for configuring the SoftEther VPN Client. The first option is to export a profile using <b>AccountExport</b> from a previously configured client then import using <b>AccountImport</b>. The second one is to connect using the Windows SoftEther VPN Client Manager GUI using either a Windows host or wine. The wine option installs a bunch of dependencies and is not 100% functional but it does work."
%}

### Connection Script

The connection process is as follows:

1. Create static route to SoftEther VPN server via the default gateway interface
1. Start SoftEther client using `/usr/local/vpnclient/vpnclient start`
1. Bring up VPN interface and utilize DHCP

To disconnect, the process is reversed to restore prior connection.

In the script below:
* ip = vpnserver Internet IP address to DNAT (firewall)
* nicdev = workstation's physical interface
* nicgw = workstation's initial default gateway interface

I would advise walking through the script manually via bash, then tailor the script below to meet your use case. When using bash, you can still use the variables by executing `ip=216.50.190.90`[enter]. You can see the results using `echo $ip`[enter]. **Also, I was receiving messages on console from auditd (SELinux) when executing dhclient. I need to research further.**

Create and edit the file "vpn."

```
[john@wss temp]$ sudo touch /usr/local/vpnclient/vpn
[john@wss temp]$ sudo chmod +x /usr/local/vpnclient/vpn
[john@wss temp]$ sudo vi /usr/local/vpnclient/vpn
```

Copy and paste then update the bash script to fit your environment.

```
##!/bin/bash                                                                                         
# SoftEther VPN Client start/stop script                                                             

# Use DNS or DDNS for the SoftEther VPN Server                                                       
host=sevpn.mydomain.net

# Workstation interface device name from ifconfig or ip addr
nicdev=eth0

# Use SoftEther VPN Client interface; vpn_[NicCreate]
sedev=vpn_sev0

# START VPN CLIENT
if [[ $1 == "start" ]]; then

    # Determine IP address for SoftEther VPN Server
    ip=$(dig +short $host)

    # Write IP address to file for /vpn stop
    echo "$ip" > /tmp/sevpnip

    # Determine default route for workstation prior to establishing VPN
    nicgw=$(ip -4 route list 0/0 | cut -d ' ' -f 3)

    # Write workstation default gateway to file for /vpn stop
    echo "$nicgw" > /tmp/sevpngw

    # Create static host route to SoftEther VPN Server
    ip route add $ip via $nicgw dev $nicdev
    sleep .5

    # Start SoftEther VPN
    /usr/local/vpnclient/vpnclient start
    sleep .5

    # Request DHCP
    dhclient $sedev

    # Delete default route. Note on subsequent stops, the routes drop
    # automatically, thus sending to /dev/null to silence possible
    # RTNETLINK no such process error.
    ip route del default via $nicgw &>/dev/null
    sleep .5

# STOP VPN CLIENT
elif [[ $1 == "stop" ]]; then

    # Release DHCP
    dhclient -r $sedev
    sleep .5

    # Stop SoftEther VPN Client which remove interface and default route
    /usr/local/vpnclient/vpnclient stop
    sleep .5

    # Read in workstation physical network interface's default gateway
    nicgw=$(cat /tmp/sevpngw)
    sleep .5

    # Re-enstate default route
    ip route add default via $nicgw
    sleep .5

    # Read in SoftEther VPN Server IP address
    ip=$(cat /tmp/sevpnip)

    # Delete the host route to SoftEther VPN Server
    ip route del $ip

    # Cleanup
    rm -f /tmp/sevpnip
    rm -f /tmp/sevpngw

# User forgot to provide start or stop arguments
else
    echo "Please use args start or stop!"
fi
```

{% include tip.html content="point-to-point<br/>
If your intent is to use SoftEther VPN as an always-on or point-to-point VPN solution, a systemd script like https://github.com/SoftEtherVPN/SoftEtherVPN/pull/180/commits/525348b6d168ced42d8e723033bd2084f6a1eea6 is an ideal alternative."
%}

## Done!?

At this point, you should have a Secure Remote Access solution using SoftEther VPN Server & Client 4.2 on Linux. Once you have a working solution and some time with it up and running, I advise updating the virtual hub, default is vpn, to enable the "No Enumerate to Anonymous Users" to obfuscate the service.

## oVirt & Promiscuous Mode

The SoftEther project page calls out that running the SoftEther VPN Server's network interface in promiscuous mode will improve performance. I have not validated the statement. If using oVirt, you have a bit of work to permit an oVirt guest to use promiscuous mode.

First, install "vdsm-hook-macspoof" package on each Compute host.

```
[root@node1 ~]# yum install vdsm-hook-macspoof
```

Results

```
Dependencies Resolved

================================================================================
 Package                Arch       Version             Repository          Size
================================================================================
Installing:
 vdsm-hook-macspoof     noarch     4.17.32-1.el7       centos-ovirt36      21 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 21 k
Installed size: 5.1 k
Is this ok [y/d/N]: y
```

Second, update the hosted engine to have a user defined property for oVirt guests.

```
Using username "root".
root@eng's password:
Last login: Thu Aug 11 05:20:58 2016
[root@eng ~]# sudo engine-config -s "UserDefinedVMProperties=macspoof=^(true|false)$"
Please select a version:
1. 3.0
2. 3.1
3. 3.2
4. 3.3
5. 3.4
6. 3.5
7. 3.6
7
[root@eng ~]#
```

Lastly, restart the hosted engine.

{% include links.html %}
