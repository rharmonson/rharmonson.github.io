---
title: CentOS 7 Installation Guide
published: true
tags: [linux, centos, sysadmin, virtualization]
keywords: firewalld, NetworkManager, iptables, cloud-init, spacewalk, ovirt epel
last_updated: November 4, 2017
summary: "The purpose of this guide is provide the steps to install and configure a standardized CentOS 7 (7.4.1708) also known as Red Hat Enterprise Linux (RHEL) x86_64 base operating system for use with virtualization platforms."
sidebar: cos7inst_sidebar
permalink: cos7inst.html
folder: 2017
toc: false
commentIssueId: 3
---

The purpose of this guide is provide the steps to install and configure a standardized CentOS 7 (aka RHEL) x86_64 base operating system. In addition, there are several optional sections to prepare the build for use with virtualization platforms.

Current CentOS-7 Release Notes can be found at [CentOS-7 (1708) Release notes](https://wiki.centos.org/Manuals/ReleaseNotes/CentOS7).

CentOS FAQ can be found at [Questions about CentOS-7](http://wiki.centos.org/FAQ/CentOS7).

## Obtain Media

If you are new to Linux or new to CentOS minimal installations, I would advise reviewing all the information at the URL below. For this article, I am using x86_64 version, also, known as 64 bit.

* [Download CentOS Linux ISO Images](http://wiki.centos.org/Download)

Select the media of your choice to download.

I use the NetInstall installation media but this guide applies to all CentOS 7 installation media. The Minimal installation media works well, too. The primary advantage of the NetInstall is that the packages installed are the current packages and no updates are needed. It will increase the time to install over using the other installation medias due to downloading all packages versus only updates.

The NetInstall ISO installer has only the necessary bits to boot a very basic operating system then using http or ftp to download the packages to be installed. This differs from the other installation methods that use the local repository found on the installation media.

In addition, the NetInstall needs an enabled network interface prior to providing a repository URL. Generally, pick a mirror repository that is closest to your geographic location. The list of mirrors is found here:

* [List of CentOS Mirrors](https://www.centos.org/download/mirrors)

I, typically, use ftp due to its efficiency over http, but if privacy is a concern use https. The entered URL should look similar to:

* ftp://mirrors.ocf.berkeley.edu/centos/7/os/x86_64/

## Install

Boot from media and, generally, accept the defaults. You have an opportunity to provide time zone, a host name, configure network interfaces, provide DNS IP addresses, domain search, etc. If configured at installation, the resulting installation uses them. It is a time saver, however, I am going to provide the commands to configure on the assumption changes will be needed.

Two items to consider disabling at install are:

* [KDUMP](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kdump-ppc)
* [Red Hat Security Policy](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-security-policy-x86)
* [SCAP Home](https://scap.nist.gov/index.html)

The use of SCAP profiles can significantly improve the security stance of a host, thus an organization. Advise researching prior to using SCAP.

## Host Name

View current host name

```
[root@localhost ~]# hostnamectl
   Static hostname: localhost.localdomain
         Icon name: computer-vm
           Chassis: vm
        Machine ID: c868e2da40fb4e3da47ab9189916ca7a
           Boot ID: e663afbf6d9d4852a260d69bd173e5c3
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-693.el7.x86_64
      Architecture: x86-64
```

Set the hostname

```
[root@localhost ~]# hostnamectl set-hostname myhost.mydomain.net
```

Results

```
[root@localhost ~]# hostnamectl
   Static hostname: myhost.mydomain.net
         Icon name: computer-vm
           Chassis: vm
        Machine ID: c868e2da40fb4e3da47ab9189916ca7a
           Boot ID: e663afbf6d9d4852a260d69bd173e5c3
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-693.el7.x86_64
      Architecture: x86-64
```

## Network

### Network Manager

Red Hat has been changing how networking is configured and managed with an emphasis on the use of Network Manager. Network Manager is installed and in use by default on CentOS 7. Configure using either `nmtui` or `nmcli`. `nmtui` has a very intuitive interface but `nmcli` is useful for scripting.

If you have multiple interfaces, connect an Ethernet cable to the desired port, then execute `ip addr` to identify the interface. If using DHCP, it will show an IP address assigned. If not using DHCP, you should see 'up' status. Execute `nmtui` and "Edit" the interface then using `nmtui`, again, to "Activate".

### Removing Network Manager

For virtual servers, my preference is use the virtualization platform configuration management tools to manage network interfaces not Network Manager.

Begin by stopping and disabling NetworkManager

```
[root@myhost ~]# systemctl stop NetworkManager && systemctl disable NetworkManager
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
```

Now remove NetworkManager

```
[root@myhost ~]# yum -y remove NetworkManager NetworkManager-libnm
```

Results

```
================================================================================
 Package                   Arch        Version             Repository      Size
================================================================================
Removing:
 NetworkManager            x86_64      1:1.8.0-9.el7       @anaconda      4.7 M
 NetworkManager-libnm      x86_64      1:1.8.0-9.el7       @anaconda      5.6 M
Removing for dependencies:
 NetworkManager-team       x86_64      1:1.8.0-9.el7       @anaconda       53 k
 NetworkManager-tui        x86_64      1:1.8.0-9.el7       @anaconda      240 k
 NetworkManager-wifi       x86_64      1:1.8.0-9.el7       @anaconda      144 k

Transaction Summary
================================================================================
Remove  2 Packages (+3 Dependent packages)

Installed size: 11 M
Is this ok [y/N]:
```

### Hand Crafting `ifcfg` Files

By default, the CentOS installation will have created ifcfg files for detected interfaces. Backup the original files with the exception of `ifcfg-lo` which will remain unmodified. Note that all files starting with "ifcfg" within network-scripts will be processed at start of the network service unless appending `.orig`. When backing up the files, either place in a different directory or append `.orig`.

#### View Interfaces

Connect the interface to be configured and use `ip addr` identify the 'up' interface if using more than one interface.

For example with DHCP

```
[root@myhost ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:1a:4a:16:01:66 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.109/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 3431sec preferred_lft 3431sec
    inet6 fe80::21a:4aff:fe16:166/64 scope link
       valid_lft forever preferred_lft forever
```

Note interface eth0 is in an `UP` state. This is the interface to be configured. The example only has one interface, however, additional interfaces without a network cable would not show an "UP" state.

#### Configure Interface

Create or edit a configuration file using `vi /etc/sysconfig/network-scripts/ifcfg-eth0` and replace the values given in the example below with yours; IPADDR, NETMASK, and GATEWAY. The entry "DEFROUTE=yes" assumes the interface is to be the default route for unknown routes. All other interfaces should have "DEFROUTE=no." You may find many more values within the original ifcfg files for use with NetworkManager. If not using NetworkManager, these can be safely removed.

Original `ifcfg-eth0`

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="eth0"
UUID="1f655fcc-6cbd-4de4-903c-124b1b0e51b1"
DEVICE="eth0"
ONBOOT="yes"
```

New ifcfg-eth0 without NetworkManger entries and set to static IP address.

```
TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=yes
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.2
NETMASK=255.255.255.0
GATEWAY=192.168.1.254
```

After saving the ifcfg file, restart network services.

```
[root@myhost ~]# systemctl restart network
```

Results

```
[root@myhost~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:1a:4a:16:01:66 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.2/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 3431sec preferred_lft 3431sec
    inet6 fe80::21a:4aff:fe16:166/64 scope link
       valid_lft forever preferred_lft forever
```

{% include note.html content="Additional Values<br/>
1. `NM_MANAGED=no` disables Network Manager for an interface, if using Network Manager<br/>
2. `IPV6INIT=no` disables IPv6 for an interface<br/>
3. `DEFROUTE=no` or `DEFROUTE=yes` excludes or sets an interface as the default route, respectively, if using Network Manager<br/>
4. `PEERDNS=yes` adds the interface's DNS settings to the `/etc/resolv.conf`<br/>
5. `PREFIX` is an alternative to `NETMASK`"
%}

Additional interfaces if needed have a much simpler configuration.

```
[root@myhost ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth1

TYPE=Ethernet
BOOTPROTO=none
DEFROUTE=no
DEVICE=eth1
ONBOOT=yes
IPADDR=192.168.2.1
NETMASK=255.255.255.0
GATEWAY=192.168.2.254
```

If using bonds, bridges, or teams, details can be found on Red Hat's [Networking Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/index.html).

### Disable IPv6

Previously, I disabled IPv6 everywhere for my host builds. Unfortunately, developers are increasingly using the IPv6 and as a consequence, some services will break without it. To overcome this potential requirement, disable IPv6 per interfaces.

I provide instructions in the section titled "Disable IPv6 Everywhere" if you want to kill IPv6 for all interfaces as a default then enable as necessary.

View IPv6 Settings using `sysctl -a`

```
[root@myhost ~]# sysctl -a | grep -i ipv6.conf.*.disable_ipv6
sysctl: reading key "net.ipv6.conf.all.stable_secret"
sysctl: reading key "net.ipv6.conf.default.stable_secret"
sysctl: reading key "net.ipv6.conf.eth0.stable_secret"
sysctl: reading key "net.ipv6.conf.lo.stable_secret"
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.eth0.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Note the value of "0" means the feature is not enabled. Enable the "eth0" disable policy to stop the eth0 interface from using IPv6.

Edit `vi /etc/sysctl.conf` which will have no entries. We will add and enable eth0.disable for IPv6.

Results

```
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv6.conf.eth0.disable_ipv6=1
```

At this point, you can reboot or use `sysctl` to load /etc/sysctl.conf.

```
[root@myhost~]# sysctl -p
net.ipv6.conf.eth0.disable_ipv6 = 1
```

Using `ip addr` note there is no IPv6 address associated with eth0.

```
[root@myhost etc]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:1a:4a:16:01:54 brd ff:ff:ff:ff:ff:ff
    inet 192.168.131.242/24 brd 192.168.131.255 scope global dynamic eth0
       valid_lft 3047sec preferred_lft 3047sec
```

{% include tip.html content="*Disable IPv6 Everywhere*<br/>
<br/>
To disable the use of IPv6 for everything on the Linux host, enable *all* and *default* within sysctl.conf to ensure no interfaces uses IPv6.<br/>
<br/>
net.ipv6.conf.all.disable_ipv6=1<br/>
net.ipv6.conf.default.disable_ipv6=1"
%}

```
[root@myhost ~]# vi /etc/sysctl.conf
# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
```

You can enable IPv6 for a specific interface by adding it to sysctl.conf with a value of 0 to override the "all" and "default." For example, to enable the IPv6 loopback interface or `::1`:

```
net.ipv6.conf.lo.disable_ipv6 = 0
```

{% include tip.html content="*Interface ::1*<br/>
<br?>
Recently had a nasty experience with upgrading IPA server from 4.3 to 4.5. Apparently the new ipa-server-upgrade does a check and if ::1 exists in the /etc/hosts file, the upgrade implodes. Stupid!<br/>
<br/>
I am going to remove ::1 in my builds and see what breaks. I have seen evidence that a minority of developers assume IPv6 interfaces exist and don't bother to support IPv4. For $%@#!& sakes why?"
%}

### NOZEROCONF

Does anyone use Zero Configuration Networking? I do not! Add the following line to `/etc/sysconfig/network` to prevent zero configuration networking, i.e. 169.254.0.0/16 in the absence of static or DHCP IP address assignment.

```
NOZEROCONF=yes
```

Read more about [Zeroconf](http://www.zeroconf.org/).

### Name Resolution

Network Manager or DHClient may have updated resolv.conf to reflect ifcfg's DNS1, DNS2, and DOMAIN settings. If not, `vi /etc/resolv.conf` and update appropriately. Mine is given below.

```
[root@myhost ~]# cat /etc/resolv.conf
search mydomain.net
nameserver 65.6.64.6
nameserver 65.6.65.6
```

### Network Testing

Use ping to verify basic interface, routing, and name resolution operation.

```
[root@myhost etc]# ping www.google.com -c 5
PING www.google.com (216.58.194.196) 56(84) bytes of data.
64 bytes from sfo03s01-in-f4.1e100.net (216.58.194.196): icmp_seq=1 ttl=53 time=19.0 ms
64 bytes from sfo03s01-in-f4.1e100.net (216.58.194.196): icmp_seq=2 ttl=53 time=17.5 ms
64 bytes from sfo03s01-in-f4.1e100.net (216.58.194.196): icmp_seq=3 ttl=53 time=17.8 ms
64 bytes from sfo03s01-in-f4.1e100.net (216.58.194.196): icmp_seq=4 ttl=53 time=19.3 ms
64 bytes from sfo03s01-in-f4.1e100.net (216.58.194.196): icmp_seq=5 ttl=53 time=18.0 ms

--- www.google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 17.583/18.356/19.327/0.716 ms
```

## firewalld & iptables

As with NetworkManager, I see no compelling reason for firewalld, so my preference is to remove it and use iptables directly.

### Remove firewalld

```
[root@myhost etc]# systemctl stop firewalld && systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@myhost etc]# yum -y remove firewalld firewalld-filesystem
Loaded plugins: fastestmirror
Resolving Dependencies
--> Running transaction check
---> Package firewalld.noarch 0:0.4.4.4-6.el7 will be erased
---> Package firewalld-filesystem.noarch 0:0.4.4.4-6.el7 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                   Arch        Version             Repository      Size
================================================================================
Removing:
 firewalld                 noarch      0.4.4.4-6.el7       @anaconda      1.8 M
 firewalld-filesystem      noarch      0.4.4.4-6.el7       @anaconda      239

Transaction Summary
================================================================================
Remove  2 Packages

Installed size: 1.8 M
Is this ok [y/N]:
```

### Install iptables-services

With the removal of firewalld, use iptables-services to manage iptables and ip6tables services.

```
# [root@myhost etc]# yum -y install iptables-services
```

Results

```
================================================================================
 Package                Arch        Version                  Repository    Size
================================================================================
Installing:
 iptables-services      x86_64      1.4.21-18.2.el7_4        updates       51 k
Updating for dependencies:
 iptables               x86_64      1.4.21-18.2.el7_4        updates      428 k

Transaction Summary
================================================================================
Install  1 Package
Upgrade             ( 1 Dependent package)

Total download size: 479 k
Is this ok [y/d/N]:
```

Note the iptables package in the results above is due to an update being available. If you have the current package, it will not be displayed in the results.

Enable and start iptables-services.

```
[root@myhost etc]# systemctl enable iptables ip6tables && systemctl start iptables ip6tables
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
Created symlink from /etc/systemd/system/basic.target.wants/ip6tables.service to /usr/lib/systemd/system/ip6tables.service.
```

## Firewall Policies

There are two sets of files for iptables and ip6tables that set policy and configuration found in `/etc/sysconfig`. The files are:

1. iptables
1. ip6tables
1. iptables-config
1. ip6tables-config

I would advise reviewing both sets of file, but for the purpose of this guide, we will focus on iptables and ip6tables.

The iptables default:

```
[root@myhost sysconfig]# cat iptables
# sample configuration for iptables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

Results

```
[root@myhost sysconfig]# iptables -L -nv
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
  326 24146 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0   
    0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0   
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22
    5   692 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT 195 packets, 30476 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

and the ip6tables default:

```
[root@myhost sysconfig]# cat ip6tables
# sample configuration for ip6tables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p ipv6-icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -d fe80::/64 -p udp -m udp --dport 546 -m state --state NEW -j ACCEPT
-A INPUT -j REJECT --reject-with icmp6-adm-prohibited
-A FORWARD -j REJECT --reject-with icmp6-adm-prohibited
COMMIT
```

Results

```
[root@myhost sysconfig]# ip6tables -L -nv
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all      *      *       ::/0                 ::/0                 state RELATED,ESTABLISHED
    0     0 ACCEPT     icmpv6    *      *       ::/0                 ::/0       
    0     0 ACCEPT     all      lo     *       ::/0                 ::/0        
    0     0 ACCEPT     tcp      *      *       ::/0                 ::/0                 state NEW tcp dpt:22
    0     0 ACCEPT     udp      *      *       ::/0                 fe80::/64            udp dpt:546 state NEW
    0     0 REJECT     all      *      *       ::/0                 ::/0                 reject-with icmp6-adm-prohibited

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REJECT     all      *      *       ::/0                 ::/0                 reject-with icmp6-adm-prohibited

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

In reviewing the firewall policies, we are permitting established connections, incoming IPv4 and IPv6 SSH connections, incoming IPv4 ICMP, IPv6 DHCP, then rejecting AND communicating anything not explicitly permitted using icmp-host-prohibited. If you are satisfied with the defaults, skip to the next section. Otherwise, backup the existing policies.

```
[root@myhost sysconfig]# cd ~
[root@myhost ~]# iptables-save > ip4-backup.fw
[root@myhost ~]# ip6tables-save > ip6-backup.fw
[root@myhost ~]# ll
total 12
-rw-------. 1 root root 1296 Nov  4 22:08 anaconda-ks.cfg
-rw-r--r--. 1 root root  471 Nov  5 00:28 ip4-backup.fw
-rw-r--r--. 1 root root  551 Nov  5 00:28 ip6-backup.fw
```

Copy to new files for editing.

```
[root@myhost ~]# cp ip4-backup.fw ip4-default.fw
[root@myhost ~]# cp ip6-backup.fw ip6-default.fw
```

Update ip4-defaults.fw

```
[root@myhost ~]# cat ip4-default.fw
# IPv4 Default Polcies
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
COMMIT
```

Update ip6-defaults.fw

```
cat ip
[root@myhost ~]# cat ip6-default.fw
# IPv6 Default Policies
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT DROP [0:0]
COMMIT
```

Before committing changes, use `iptables-restore` and `ip6tables-restore` to test (-t) and load then review.

```
[root@myhost ~]# iptables-restore -t ip4-default.fw
[root@myhost ~]# iptables-restore ip4-default.fw
[root@myhost ~]# ip6tables-restore -t ip6-default.fw
[root@myhost ~]# ip6tables-restore ip6-default.fw
```

Results of ip4-default.fw

```
[root@myhost ~]# iptables -L -nv
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
  133 11528 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8
    0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0   
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            state NEW tcp dpt:22

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 100 packets, 10624 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

Results of ip6-default.fw

```
[root@myhost ~]# ip6tables -L -nv
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

If satisfied with the new policies, save the running firewall policies using the service command which uses iptables-save to /etc/sysconfig/iptables.

```
[root@myhost ~]# service iptables save
[root@myhost ~]# service ip6tables save
```

To permit additional services, insert additional rules using -A under the SSH rule. For example:

```
# IPv4 Default Polcies
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
COMMIT
```

{% include tip.html content="
As you update ip4-default.fw to ip4-serviceX.fw, use iptables multiport feature to simply access control lists. For example, a host providing web, kerberos, DNS, NTP, and LDAP services can configured using:<br/>
<br/>
-I INPUT -p udp -m multiport --dports 88,464,53,123 -j ACCEPT<br/>
-I INPUT -p tcp -m multiport --dports 80,443,389,636,88,464,53 -j ACCEPT"
%}

## SELinux

As much as I love the idea of SELinux, the reality is that developers as a whole have not adopted its use. As a consequence, I have lost far too many hours trouble shooting installation failures to identify SELinux as the culprit. I have not given up entirely on SELinux, but I would advise setting it to "permissive" when installing new services and testing. Once testing is complete, set SELinux to "enforcing" and test again.

Update the SELinux config file `/etc/selinux/config` to `SELINUX=permissive`.

```
# sed -i 's/=enforcing/=permissive/g' /etc/selinux/config
```

Execute `setenforce 0` to set the current session to permissive or reboot to utilize the updated config.

## Time zone

After installation, the default time zone is America/New_York. CentOS 7 uses `timedatectl` to manage time and date related settings.

Check current settings using `timedatectl`

```
[root@myhost ~]# timedatectl
      Local time: Sun 2017-11-05 01:09:12 EDT
  Universal time: Sun 2017-11-05 05:09:12 UTC
        RTC time: Sun 2017-11-05 05:09:12
       Time zone: America/New_York (EDT, -0400)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2017-03-12 01:59:59 EST
                  Sun 2017-03-12 03:00:00 EDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2017-11-05 01:59:59 EDT
                  Sun 2017-11-05 01:00:00 EST
```

Find your time zone

```
[root@myhost ~]# timedatectl list-timezones | grep -i angeles
America/Los_Angeles
```

Set your time zone

```
[root@myhost ~]# timedatectl set-timezone America/Los_Angeles
```

Results

```
[root@myhost ~]# timedatectl
      Local time: Sat 2017-11-04 22:10:20 PDT
  Universal time: Sun 2017-11-05 05:10:20 UTC
        RTC time: Sun 2017-11-05 05:10:20
       Time zone: America/Los_Angeles (PDT, -0700)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2017-03-12 01:59:59 PST
                  Sun 2017-03-12 03:00:00 PDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2017-11-05 01:59:59 PDT
                  Sun 2017-11-05 01:00:00 PST
```

## Time & Date

Set the current local time and date using `timedatectl`.

```
[root@myhost ~]# timedatectl set-time "2017-11-04 22:17:00"
```

Results

```
[root@myhost ~]# timedatectl
      Local time: Sat 2017-11-04 22:17:01 PDT
  Universal time: Sun 2017-11-05 05:17:01 UTC
        RTC time: Sun 2017-11-05 05:17:01
       Time zone: America/Los_Angeles (PDT, -0700)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2017-03-12 01:59:59 PST
                  Sun 2017-03-12 03:00:00 PDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2017-11-05 01:59:59 PDT
                  Sun 2017-11-05 01:00:00 PST
```

{% include note.html content="
When using `timedatectl set-time you may` you may receive an error `Failed to set time: Automatic time synchronization is enabled` if chronyd is in use. If the system time or date is off, see the next section."
%}

### chronyd

To update chrony time sources, edit the top of `/etc/chrony.conf` with your preferred time sources.

From

```
[root@myhost ~]# cat /etc/chrony.conf
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
server 192.168.101.252
server 192.168.101.253

```

Restart chronyd

```
[root@myhost ~]# systemctl restart chronyd
```

### ntpd

An alternative to chrony is ntp. To update ntp time sources, edit  `/etc/ntp.conf` with your preferred time sources.

From

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
server 192.168.101.252
server 192.168.101.253
```

Restart ntpd

```
[root@myhost ~]# systemctl restart ntpd
```

## EPEL (optional)
To install Extra Packages for Linux (EPEL), simply install the package from the CentOS base repository.

```
[root@myhost~]# yum -y install epel-release
```

Results

```
================================================================================
 Package                Arch             Version         Repository        Size
================================================================================
Installing:
 epel-release           noarch           7-9             extras            14 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 14 k
Installed size: 24 k
Is this ok [y/d/N]:
```

## oVirt 4 Guest (optional)

oVirt is a blend of "Open" and "Virtualization" and is the upstream project for Red Hat Enterprise Virtualization (RHEV). If using CentOS 7 as an oVirt Guest (virtual machine), install your version of the oVirt repository and guest agent. I am using oVirt 4.1.

### oVirt Repository

```
[root@myhost ~]# yum -y install http://resources.ovirt.org/pub/yum-repo/ovirt-release41.rpm
```

Results

```
================================================================================
 Package            Arch      Version                 Repository           Size
================================================================================
Installing:
 ovirt-release41    noarch    4.1.6-1.el7.centos      /ovirt-release41     10 k

Transaction Summary
================================================================================
Install  1 Package

Total size: 10 k
Installed size: 10 k
Is this ok [y/d/N]:
```

### oVirt Guest

```
[root@myhost ~]# yum -y install ovirt-guest-agent-common
```

Results

```
================================================================================
 Package                       Arch        Version              Repository
                                                                           Size
================================================================================
Installing:
 ovirt-guest-agent-common      noarch      1.0.13-2.el7         epel       70 k
Installing for dependencies:
 libnl                         x86_64      1.1.4-3.el7          base      128 k
 python-ethtool                x86_64      0.8-5.el7            base       33 k
 usermode                      x86_64      1.111-5.el7          base      193 k

Transaction Summary
================================================================================
Install  1 Package (+3 Dependent packages)

Total download size: 424 k
Installed size: 1.4 M
Is this ok [y/d/N]:
```

The ovirt-guest-agent.server will start on next boot or `systemctl start ovirt-guest-agent.service`.

## CloudInit (optional)

CloudInit handles early initialization of virtual machines. I use the cloud-init service with oVirt Engine to configure network settings, passwords, and other settings when initializing from virtual machines templates.

{% include warning.html content="
CentOS 7.4.1708 at release the cloud-init 0.7.9 is *broken*. I have seen number of different bug reports and solutions but no working package has been release as of November 5, 2017. If you have an existing installation using cloud-init 0.7.5 from CentOS 7.3 use the yum-plugin-versionlock to lock the version. Version 0.7.5 appears to be working without issue on 7.4."
%}

```
[root@myhost~]# yum -y install cloud-init
```

Results

```
================================================================================
 Package                Arch   Version               Repository            Size
================================================================================
Installing:
 cloud-init             x86_64 0.7.9-9.el7.centos.2  base                 622 k
Installing for dependencies:
 PyYAML                 x86_64 3.10-11.el7           base                 153 k
 audit-libs-python      x86_64 2.7.6-3.el7           base                  73 k
 checkpolicy            x86_64 2.5-4.el7             base                 290 k
 libcgroup              x86_64 0.41-13.el7           base                  65 k
 libsemanage-python     x86_64 2.5-8.el7             base                 104 k
 libyaml                x86_64 0.1.4-11.el7_0        base                  55 k
 net-tools              x86_64 2.0-0.22.20131004git.el7
                                                     base                 305 k
 policycoreutils-python x86_64 2.5-17.1.el7          base                 446 k
 pyserial               noarch 2.6-6.el7             base                 124 k
 python-IPy             noarch 0.75-6.el7            base                  32 k
 python-babel           noarch 0.9.6-8.el7           base                 1.4 M
 python-backports       x86_64 1.0-8.el7             base                 5.8 k
 python-backports-ssl_match_hostname
                        noarch 3.4.0.2-4.el7         base                  12 k
 python-chardet         noarch 2.2.1-1.el7_1         base                 227 k
 python-jinja2          noarch 2.7.2-2.el7           base                 515 k
 python-jsonpatch       noarch 1.2-4.el7             base                  15 k
 python-jsonpointer     noarch 1.9-2.el7             base                  13 k
 python-markupsafe      x86_64 0.11-10.el7           base                  25 k
 python-prettytable     noarch 0.7.2-3.el7           base                  37 k
 python-requests        noarch 2.6.0-1.el7_1         base                  94 k
 python-setuptools      noarch 0.9.8-7.el7           base                 397 k
 python-urllib3         noarch 1.10.2-3.el7          base                 101 k
 python2-six            noarch 1.10.0-9.el7          ovirt-centos-ovirt41  31 k
 setools-libs           x86_64 3.3.8-1.1.el7         base                 612 k

Transaction Summary
================================================================================
Install  1 Package (+24 Dependent packages)

Total download size: 5.6 M
Installed size: 22 M
Is this ok [y/d/N]:
```

The cloud-init.server will start on next boot or `systemctl start cloud-init.service`.

Reference: [Using Cloud-Init to automate the configuration of VMs](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.0/html/virtual_machine_management_guide/sect-using_cloud-init_to_automate_the_configuration_of_virtual_machines)

## Spacewalk Client (optional)

Spacewalk is the upstream project for Satellite 5 aka Satellite Classic. I use it for patch and configuration management and my guide to build the Spacewalk 2.6 Server can be found at [Spacewalk 2.6](https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Configuration-and-Patch-Management-with-Spacewalk-2.6-on-CentOS-7.3.1611-Minimal).

### Spacewalk Client Repository

Install the repository's package matching the Spacewalk server. Note the EPEL repository is a prerequisite.

```
[root@myclient ~]# yum -y install http://yum.spacewalkproject.org/2.7-client/RHEL/7/x86_64/spacewalk-client-repo-2.7-2.el7.noarch.rpm
```

Results

```
================================================================================
 Package         Arch   Version   Repository                               Size
================================================================================
Installing:
 spacewalk-client-repo
                 noarch 2.7-2.el7 /spacewalk-client-repo-2.7-2.el7.noarch 493

Transaction Summary
================================================================================
Install  1 Package

Total size: 493
Installed size: 493
Is this ok [y/d/N]:
```

### Spacewalk Client Packages

Spacewalk client packages to register and utilize core Spacewalk features like registration.

```
[root@myhost~]#  yum -y install m2crypto rhn-check rhn-client-tools rhn-setup rhnsd yum-rhn-plugin
```

Results

```
================================================================================
 Package              Arch       Version             Repository            Size
================================================================================
Installing:
 m2crypto             x86_64     0.21.1-17.el7       base                 429 k
 rhn-check            noarch     2.7.16-1.el7        spacewalk-client      60 k
 rhn-client-tools     noarch     2.7.16-1.el7        spacewalk-client     485 k
 rhn-setup            noarch     2.7.16-1.el7        spacewalk-client      96 k
 rhnsd                x86_64     5.0.30-1.el7        spacewalk-client      47 k
 yum-rhn-plugin       noarch     2.7.7-1.el7         spacewalk-client      85 k
Installing for dependencies:
 libgudev1            x86_64     219-42.el7_4.4      updates               83 k
 libxml2-python       x86_64     2.9.1-6.el7_2.3     base                 247 k
 pyOpenSSL            x86_64     0.13.1-3.el7        base                 133 k
 python-dmidecode     x86_64     3.12.2-1.el7        base                  85 k
 python-gudev         x86_64     147.2-7.el7         base                  18 k
 python-hwdata        noarch     1.7.3-4.el7         base                  32 k
 rhnlib               noarch     2.7.5-1.el7         spacewalk-client      69 k

Transaction Summary
================================================================================
Install  6 Packages (+7 Dependent packages)

Total download size: 1.8 M
Installed size: 7.5 M
Is this ok [y/d/N]:
```

To use Spacewalk, the host must be registered and additional functionality will require additional packages and configuration. Details on the steps to complete a Spacewalk client setup are found at the client section in my [Configuration and Patch Management Guide](https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Configuration-and-Patch-Management-with-Spacewalk-2.6-on-CentOS-7.3.1611-Minimal#spacewalk-client).

## FreeIPA Client (optional)

If joining a host to a FreeIPA Realm, configure DNS and NTP to use the FreeIPA Master and Replicas for both prior to joining. It is not a requirement per se, but the client must be able to resolve the Masters, Replicas, and a variety of service records, and Kerberos requires the client's time to be accurate in respect to the Realm's time.

### FreeIPA client package

```
[root@myhost ~]# yum install ipa-client
```

Results

```
================================================================================
 Package              Arch   Version                 Repository            Size
================================================================================
Installing:
 ipa-client           x86_64 4.5.0-21.el7.centos.2.2 updates              247 k
Installing for dependencies:
 autofs               x86_64 1:5.0.7-69.el7          base                 808 k
 bind-libs            x86_64 32:9.9.4-51.el7         updates              1.0 M
 bind-utils           x86_64 32:9.9.4-51.el7         updates              203 k
 c-ares               x86_64 1.10.0-3.el7            base                  78 k
 certmonger           x86_64 0.78.4-3.el7            base                 598 k
 cyrus-sasl-gssapi    x86_64 2.1.26-21.el7           base                  41 k
 gssproxy             x86_64 0.7.0-4.el7             base                 105 k
 hesiod               x86_64 3.2.1-3.el7             base                  30 k
 http-parser          x86_64 2.7.1-5.el7_4           updates               28 k
 ipa-client-common    noarch 4.5.0-21.el7.centos.2.2 updates              154 k
 ipa-common           noarch 4.5.0-21.el7.centos.2.2 updates              577 k
 keyutils             x86_64 1.5.8-3.el7             base                  54 k
 krb5-workstation     x86_64 1.15.1-8.el7            base                 811 k
 libbasicobjects      x86_64 0.1.1-27.el7            base                  25 k
 libcollection        x86_64 0.6.2-27.el7            base                  41 k
 libdhash             x86_64 0.4.3-27.el7            base                  28 k
 libevent             x86_64 2.0.21-4.el7            base                 214 k
 libini_config        x86_64 1.3.0-27.el7            base                  63 k
 libipa_hbac          x86_64 1.15.2-50.el7_4.6       updates              127 k
 libkadm5             x86_64 1.15.1-8.el7            base                 174 k
 libldb               x86_64 1.1.29-1.el7            base                 128 k
 libnfsidmap          x86_64 0.25-17.el7             base                  49 k
 libpath_utils        x86_64 0.2.1-27.el7            base                  27 k
 libref_array         x86_64 0.1.5-27.el7            base                  26 k
 libsmbclient         x86_64 4.6.2-11.el7_4          updates              130 k
 libsss_autofs        x86_64 1.15.2-50.el7_4.6       updates              129 k
 libsss_certmap       x86_64 1.15.2-50.el7_4.6       updates              150 k
 libsss_idmap         x86_64 1.15.2-50.el7_4.6       updates              132 k
 libsss_nss_idmap     x86_64 1.15.2-50.el7_4.6       updates              130 k
 libsss_sudo          x86_64 1.15.2-50.el7_4.6       updates              127 k
 libtalloc            x86_64 2.1.9-1.el7             base                  33 k
 libtdb               x86_64 1.3.12-2.el7            base                  47 k
 libtevent            x86_64 0.9.31-1.el7            base                  36 k
 libtirpc             x86_64 0.2.4-0.10.el7          base                  88 k
 libverto-tevent      x86_64 0.2.5-4.el7             base                 9.0 k
 libwbclient          x86_64 4.6.2-11.el7_4          updates              104 k
 nfs-utils            x86_64 1:1.3.0-0.48.el7_4      updates              398 k
 oddjob               x86_64 0.31.5-4.el7            base                  69 k
 oddjob-mkhomedir     x86_64 0.31.5-4.el7            base                  38 k
 psmisc               x86_64 22.20-15.el7            base                 141 k
 pyOpenSSL            x86_64 0.13.1-3.el7            base                 133 k
 python-cffi          x86_64 1.6.0-5.el7             base                 218 k
 python-dateutil      noarch 1:2.4.2-1.el7           ovirt-centos-ovirt41  83 k
 python-dns           noarch 1.12.0-4.20150617git465785f.el7
                                                     base                 233 k
 python-enum34        noarch 1.0.4-1.el7             base                  52 k
 python-gssapi        x86_64 1.2.0-3.el7             base                 322 k
 python-idna          noarch 2.4-1.el7               base                  94 k
 python-ipaddress     noarch 1.0.16-2.el7            base                  34 k
 python-jwcrypto      noarch 0.2.1-1.el7             base                  41 k
 python-ldap          x86_64 2.4.15-2.el7            base                 159 k
 python-libipa_hbac   x86_64 1.15.2-50.el7_4.6       updates              120 k
 python-netaddr       noarch 0.7.5-7.el7             base                 983 k
 python-netifaces     x86_64 0.10.4-3.el7            base                  17 k
 python-nss           x86_64 0.16.0-3.el7            base                 266 k
 python-ply           noarch 3.4-11.el7              base                 123 k
 python-pycparser     noarch 2.14-1.el7              base                 104 k
 python-qrcode-core   noarch 5.0.1-1.el7             base                  40 k
 python-sss-murmur    x86_64 1.15.2-50.el7_4.6       updates              110 k
 python-sssdconfig    noarch 1.15.2-50.el7_4.6       updates              153 k
 python-yubico        noarch 1.2.3-1.el7             base                  47 k
 python2-cryptography x86_64 1.7.2-1.el7_4.1         updates              502 k
 python2-ipaclient    noarch 4.5.0-21.el7.centos.2.2 updates              641 k
 python2-ipalib       noarch 4.5.0-21.el7.centos.2.2 updates              645 k
 python2-pyasn1       noarch 0.1.9-7.el7             base                 100 k
 python2-pyasn1-modules
                      noarch 0.1.9-7.el7             base                  59 k
 pyusb                noarch 1.0.0-0.11.b1.el7       base                  66 k
 quota                x86_64 1:4.01-14.el7           base                 179 k
 quota-nls            noarch 1:4.01-14.el7           base                  90 k
 rpcbind              x86_64 0.2.0-42.el7            base                  59 k
 samba-client-libs    x86_64 4.6.2-11.el7_4          updates              4.7 M
 samba-common         noarch 4.6.2-11.el7_4          updates              197 k
 sssd                 x86_64 1.15.2-50.el7_4.6       updates              119 k
 sssd-ad              x86_64 1.15.2-50.el7_4.6       updates              225 k
 sssd-client          x86_64 1.15.2-50.el7_4.6       updates              185 k
 sssd-common          x86_64 1.15.2-50.el7_4.6       updates              1.3 M
 sssd-common-pac      x86_64 1.15.2-50.el7_4.6       updates              181 k
 sssd-ipa             x86_64 1.15.2-50.el7_4.6       updates              317 k
 sssd-krb5            x86_64 1.15.2-50.el7_4.6       updates              158 k
 sssd-krb5-common     x86_64 1.15.2-50.el7_4.6       updates              192 k
 sssd-ldap            x86_64 1.15.2-50.el7_4.6       updates              226 k
 sssd-proxy           x86_64 1.15.2-50.el7_4.6       updates              154 k
 tcp_wrappers         x86_64 7.6-77.el7              base                  78 k
 xmlrpc-c             x86_64 1.32.5-1905.svn2451.el7 base                 130 k
 xmlrpc-c-client      x86_64 1.32.5-1905.svn2451.el7 base                  32 k

Transaction Summary
================================================================================
Install  1 Package (+84 Dependent packages)

Total download size: 21 M
Installed size: 75 M
Is this ok [y/d/N]:
```

### Join FreeIPA Realm

Execute the `ipa-client-install` command and provide privileged credentials, admin, to join.

```
$ sudo ipa-client-install --force-ntpd --enable-dns-updates --mkhomedir
```

## Update

Update the host.

```
[root@myhost ~]# yum clean all
[root@myhost ~]# yum -y update yum
[root@myhost ~]# yum -y update
```

## VM Template

If using the CentOS host created using the instructions above as a virtual machine (VM) template, I use the following process to prepare the VM.

1. Clean yum
1. Clear machine-id
1. Enable cloud-init
1. Delete SSH host keys
1. Delete history and logs
1. sys-unconfig
1. Convert virtual machine to template
1. Deploy virtual machine using template

To simplify the process further, create a file, set it as executable and paste the following. Execute using `./sealvm.sh` as the last step in the template build process. Alternatively, you can `bash < sealmv.sh`.

```
[root@myhost~]# touch sealvm.sh
[root@myhost~]# chmod +x sealvm.sh
[root@myhost~]# vi sealvm.sh
```

cut+paste

```
#!/bin/bash
# Seal Virtual Machine

yum clean all
systemctl enable cloud-init
> /etc/machine-id
rm -f /etc/ssh/ssh_host_*
rm -rf /root/.ssh/
rm -f /root/anaconda-ks.cfg
rm -f /root/.bash_history
unset HISTFILE
rm -f /var/log/boot.log
rm -f /var/log/btmp
rm -f /var/log/cron
rm -f /var/log/dmesg
rm -f /var/log/dmesg.old
rm -f /var/log/grubby
rm -f /var/log/lastlog
rm -f /var/log/maillog
rm -f /var/log/messages
rm -f /var/log/secure
rm -f /var/log/spooler
rm -f /var/log/tallylog
rm -f /var/log/wpa_supplicant.log
rm -f /var/log/wtmp
rm -f /var/log/yum.log
rm -f /var/log/audit/audit.log
rm -f /var/log/ovirt-guest-agent/ovirt-guest-agent.log
rm -f /var/log/tuned/tuned.log
sys-unconfig
```

Additional details on sealing virtual machines for oVirt is found in Red Hat's [Virtual Machine Management Guide](https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.1/html/virtual_machine_management_guide/chap-templates#sect-Sealing_Virtual_Machines_in_Preparation_for_Deployment_as_Templates).

## Done!?

The build is complete. However, you may want to consider the following:

## Disable Postfix

Depending on the purpose of the system, postfix may not be needed.

```
[root@myhost ~]# systemctl stop postfix && systemctl disable postfix
Removed symlink /etc/systemd/system/multi-user.target.wants/postfix.service.
[root@myhost ~]# yum -y remove postfix
Loaded plugins: fastestmirror
Resolving Dependencies
--> Running transaction check
---> Package postfix.x86_64 2:2.10.1-6.el7 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package         Arch           Version                 Repository         Size
================================================================================
Removing:
 postfix         x86_64         2:2.10.1-6.el7          @anaconda          12 M

Transaction Summary
================================================================================
Remove  1 Package

Installed size: 12 M
```

## Additional Packages

I install a number of optional packages for my builds including:

* deltarpm: download deltas for yum packages to reduce time and data usage
* yum-utils: variety of useful utils to query yum
* yum-versionlock: permits locking a version of packages when exclude is not an option (spacewalk)
* yum-plugin-priorities: set repository priorities when a repository is a requirement but has a history of trashing things
* yum-plugin-security: used to query and install only security packages
* rpmconf: darn useful to see what configuration files were replaced after updates
* wget: alternative to curl
* bzip2: un/archive utility for .tar.bz2 files
* tmux: alternative to screen

```
[root@myhost ~]# yum -y install deltarpm yum-utils yum-plugin-versionlock yum-plugin-priorities yum-plugin-security rpmconf wget bzip2 tmux
```

Results

```
================================================================================
 Package                    Arch       Version                Repository   Size
================================================================================
Installing:
 bzip2                      x86_64     1.0.6-13.el7           base         52 k
 deltarpm                   x86_64     3.6-3.el7              base         82 k
 rpmconf                    noarch     0.3.4-1.el7            epel         21 k
 tmux                       x86_64     1.8-4.el7              base        243 k
 wget                       x86_64     1.14-15.el7_4.1        updates     547 k
 yum-plugin-priorities      noarch     1.1.31-42.el7          base         27 k
 yum-plugin-versionlock     noarch     1.1.31-42.el7          base         32 k
 yum-utils                  noarch     1.1.31-42.el7          base        117 k
Installing for dependencies:
 libevent                   x86_64     2.0.21-4.el7           base        214 k
 libxml2-python             x86_64     2.9.1-6.el7_2.3        base        247 k
 python-kitchen             noarch     1.1.1-5.el7            base        267 k

Transaction Summary
================================================================================
Install  8 Packages (+3 Dependent packages)

Total download size: 1.8 M
Installed size: 6.8 M
Is this ok [y/d/N]:
```
{% include links.html %}
