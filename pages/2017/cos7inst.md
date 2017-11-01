---
title: CentOS 7 Installation Guide
published: true
tags: [linux, centos, sysadmin, virtualization]
keywords: firewalld, NetworkManager, iptables, cloud-init, spacewalk, ovirt epel
last_updated: July 30, 2017
summary: "The purpose of this guide is provide the steps to install and configure a standardized CentOS 7.3.1611 (aka RHEL) x86_64 base operating system."
sidebar: cos7inst_sidebar
permalink: cos7inst.html
folder: 2017
toc: false
commentIssueId: 3
---

The purpose of this guide is provide the steps to install and configure a standardized CentOS 7.3.1611 (aka RHEL) x86_64 base operating system. In addition, several optional sections prepare the installation for use with virtualization platforms.

Current CentOS-7 Release Notes can be found at https://wiki.centos.org/Manuals/ReleaseNotes/CentOS7.

CentOS FAQ can be found at http://wiki.centos.org/FAQ/CentOS7.

## Obtain Media

If you are new to Linux or new to CentOS minimal installations, I would advise reviewing all the information at the URL below. For this article, I am using x86_64 version, also, known as 64 bit.

Download: http://wiki.centos.org/Download

I use either the Minimal or NetInstall installation media. The primary advantage with the Minimal is you installation without having to exit to the Internet. The primary advantage of the NetInstall is that the packages installed are the current packages and no update is needed.


The NetInstall ISO installer has only the necessary bits to boot a very basic operating system then using http or ftp to download the packages to be installed. This differs from the other installation methods that use the local repository found on the installation media. There is no link to the NetInstall ISO on CentOS's download page. However, if select mirrors and you browse, you will find it with the other ISO installation media.

For example:

    http://mirrors.ocf.berkeley.edu/centos/7/isos/x86_64/CentOS-7-x86_64-NetInstall-1611.iso

During the install, you will need to provide a repository URL such as:

    http://mirrors.ocf.berkeley.edu/centos/7/os/x86_64/

For the Minimal installation media, click the Minimal link.

## Install

Boot from media and, generally, accept the defaults. You have an opportunity to provide time zone, a host name, configure network interfaces, provide DNS IP addresses, domain search, etc. If configured at this point, the installation script automatically configures the resulting installation using these settings. It is a time saver, however, I am going to assume these settings have not been set or changes will be needed.

## Host Name

View current host name

```
[root@localhost ~]# hostnamectl
   Static hostname: localhost.localdomain
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 35313d6a330746b787785c2db7e97ce1
           Boot ID: 4cd94f85c3ca4de6bcb4cf87c7f551f9
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-514.6.1.el7.x86_64
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
        Machine ID: 35313d6a330746b787785c2db7e97ce1
           Boot ID: 4cd94f85c3ca4de6bcb4cf87c7f551f9
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-514.6.1.el7.x86_64
      Architecture: x86-64
```

## Network

### Network Manager

Red Hat has been changing how networking is configured and managed with an emphasis on the use of Network Manager. Network Manager is installed and in use by default on CentOS 7. Configure using either `nmtui` or `nmcli`. `nmtui` has a very intuitive interface but `nmcli` is useful for scripting.

If you have multiple interfaces, connect an Ethernet cable to the desired port, then execute `ip addr` to identify the interface. If using DHCP, it will show an IP address assigned. If not using DHCP, you should see 'up' status. Execute `nmtui` and "Edit" the interface then using `nmtui`, again, to "Activate".

### Removing Network Manager

For Minimal installations of CentOS, my preference is to remove Network Manager. I see no compelling reason to use it on a server.

Begin by stopping and disabling NetworkManager

```
[root@myhost ~]# systemctl stop NetworkManager
[root@myhost ~]# systemctl disable NetworkManager
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
```

Now remove NetworkManager

```
[root@myhost ~]# yum remove NetworkManager
```

Results

```
================================================================================
 Package                  Arch        Version               Repository     Size
================================================================================
Removing:
 NetworkManager           x86_64      1:1.4.0-14.el7_3      @updates       10 M
 NetworkManager-libnm     x86_64     1:1.4.0-14.el7_3        @updates     1.1 M

Removing for dependencies:
 NetworkManager-team      x86_64      1:1.4.0-14.el7_3      @updates       53 k
 NetworkManager-tui       x86_64      1:1.4.0-14.el7_3      @updates      266 k
 NetworkManager-wifi      x86_64      1:1.4.0-14.el7_3      @updates      144 k

Transaction Summary
================================================================================
Remove  2 Package (+3 Dependent packages)

Installed size: 12 M
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
DEFROUTE=yes
DEVICE=eth1
ONBOOT=yes
IPADDR=192.168.2.1
NETMASK=255.255.255.0
GATEWAY=192.168.2.254
```

If using bonds, bridges, or teams, details can be found here:

```
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/index.html
```

Reference

```
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/ch-Network_Interfaces.html#s1-networkscripts-files
```

### Disable IPv6

Previously, I disabled IPv6 everywhere for my host builds. Unfortunately, developers are increasingly using the IPv6 and as a consequence, some services will break without it. To overcome this potential requirement, disable IPv6 for interfaces but permit its use at the kernel level. I provide instructions in the section titled "Disable IPv6 Everywhere" if you want to kill IPv6 for the entirety of the box.

View IPv6 Settings using `sysctl -a`

```
[root@myhost ~]# sysctl -a | grep -i ipv6.conf.*.disable_ipv6
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.eth0.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0

```

Note the value of "0" means the feature is not enabled. Enable the "eth0" disable policy to stop the eth0 interface from using IPv6.

Edit `vi /etc/sysctl.conf` which will have no entries. We will add and enable eth0.disable for IPv6.

Results

```
[root@myhost ~]# vi /etc/sysctl.conf
# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv6.conf.eth0.disable_ipv6=1
```

At this point, you can reboot or use `sysctl` to load /etc/sysctl.conf.

```
[root@myhost~]# sysctl --load=/etc/sysctl.conf
net.ipv6.conf.eth0.disable_ipv6 = 1
```

Using `ip addr` note there is no IPv6 address associated with eth0.

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
    inet 192.168.101.249/24 brd 192.168.101.255 scope global dynamic eth0
       valid_lft 2510sec preferred_lft 2510sec

```

{% include tip.html content="Disable IPv6 Everywhere<br/>
To disable the use of IPv6 for everything on the Linux host, enable all and default within sysctl.conf to ensure no interfaces uses IPv6.<br/>
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

```

You can enable IPv6 for a specific interface by adding it to sysctl.conf with a value of 0 to override the "all" and "default." For example, to enable the IPv6 loopback interface or `::1`:

```
net.ipv6.conf.lo.disable_ipv6 = 0
```

### NOZEROCONF

Add the following line to /etc/sysconfig/network to prevent zero configuration networking, i.e. 169.254.0.0/16 in the absence of static or DHCP IP address assignment. Ick!
```
NOZEROCONF=yes
```

### Name Resolution

Network Manager or DHClient may have updated resolv.conf to reflect ifcfg's DNS1, DNS2, and DOMAIN settings. If not, `vi /etc/resolv.conf` and update appropriately. Mine is given below.

```
[root@myhost network-scripts]# cat /etc/resolv.conf
search mydomain.net
nameserver 8.8.8.8
nameserver 8.8.4.4
```

### Network Testing

Use ping to verify basic interface, routing, and name resolution operation.

```
[root@myhost ~]# ping www.google.com -c 5
PING www.google.com (74.125.239.48) 56(84) bytes of data.
64 bytes from nuq04s19-in-f16.1e100.net (74.125.239.48): icmp_seq=1 ttl=128 time=9.83 ms
64 bytes from nuq04s19-in-f16.1e100.net (74.125.239.48): icmp_seq=2 ttl=128 time=9.05 ms
64 bytes from nuq04s19-in-f16.1e100.net (74.125.239.48): icmp_seq=3 ttl=128 time=13.4 ms
64 bytes from nuq04s19-in-f16.1e100.net (74.125.239.48): icmp_seq=4 ttl=128 time=8.40 ms
64 bytes from nuq04s19-in-f16.1e100.net (74.125.239.48): icmp_seq=5 ttl=128 time=8.25 ms

--- www.google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4070ms
rtt min/avg/max/mdev = 8.256/9.808/13.490/1.924 ms
```


## firewalld & iptables

As with NetworkManager, I see no compelling reason for firewalld. It sits on top of iptables and adds unnecessary complexity.

My preference is to remove firewalld and use iptables directly.

### Remove firewalld

```
[root@myhost~]# systemctl stop firewalld
[root@myhost~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/basic.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@myhost~]# yum remove firewalld
Loaded plugins: fastestmirror
Resolving Dependencies
--> Running transaction check
---> Package firewalld.noarch 0:0.4.3.2-8.1.el7_3 will be erased
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch          Version                   Repository       Size
================================================================================
Removing:
 firewalld        noarch        0.4.3.2-8.1.el7_3         @updates        1.7 M

Transaction Summary
================================================================================
Remove  1 Package

Installed size: 1.7 M
Is this ok [y/N]:
```

### Install iptables-services

```
================================================================================
 Package                  Arch          Version               Repository   Size
================================================================================
Installing:
 iptables-services        x86_64        1.4.21-17.el7         base         50 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 50 k
Installed size: 24 k
Is this ok [y/d/N]:
```

Enable and start iptables-services.

```
[root@myhost~]# systemctl enable iptables
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
[root@myhost~]# systemctl start iptables
[root@myhost~]# systemctl status iptables
● iptables.service - IPv4 firewall with iptables
   Loaded: loaded (/usr/lib/systemd/system/iptables.service; enabled; vendor preset: disabled)
   Active: active (exited) since Sat 2017-02-18 15:12:52 PST; 5s ago
  Process: 1405 ExecStart=/usr/libexec/iptables/iptables.init start (code=exited, status=0/SUCCESS)
 Main PID: 1405 (code=exited, status=0/SUCCESS)

Feb 18 15:12:52 myhost.mydomain.net systemd[1]: Starting IPv4 firewall with...
Feb 18 15:12:52 myhost.mydomain.net iptables.init[1405]: iptables: Applying...
Feb 18 15:12:52 myhost.mydomain.net systemd[1]: Started IPv4 firewall with ...
Hint: Some lines were ellipsized, use -l to show in full.
```

{% include warning.html content="<br/>
If you receive error <i>Failed to execute operation: Access denied</i> when using systemctl to disable firewalld, you may have disabled the option <b>Security Policy</b> during the graphical install which results with it not being installed."
%}

## Firewall Policies

Assuming the default policies are insufficient--they most assuredly are insufficient, create an iptables script to configure IPv4 policies. If IPv6 is enabled, need to create an ip6tables script as well.

Create file, `vi ip4-default.fw`

```
[root@myhost~]# touch ip4-default.fw
[root@myhost~]# chmod +x ip4-default.fw
[root@myhost~]# vi ip4-default.fw
```

copy+paste, save, then `./ip4-default.fw`

```
#!/bin/bash
# Default IPv4 Polcies

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
iptables -I INPUT -p tcp -m conntrack --ctstate NEW -m tcp --dport 22 -j ACCEPT

# Save Changes
service iptables save

# Service
systemctl restart iptables
systemctl status iptables
```

Set the file to executable using `chmod +x ip4-default.fw` then execute `./ip4-default.fw`. Review the change using `iptables -L -nv`.

Results

```
[root@myhost ~]# iptables -L -nv
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    52 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate NEW tcp dpt:22
   62  6175 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0   
    0     0 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0            icmptype 8
    1   226 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 46 packets, 7368 bytes)
 pkts bytes target     prot opt in     out     source               destination

```

If you did not disable IPv6 'everywhere,' it is probably permitting all incoming connections.

```
[root@myhost ~]# ip6tables -L -nv
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

```

As with IPv4, create a file ip6-default.fw, enter policies, set as executable, execute, and review changes.

```
[root@myhost~]# touch ip6-default.fw
[root@myhost~]# chmod +x ip6-default.fw
[root@myhost~]# vi ip6-default.fw
```

copy+paste, save, then `./ip6-default.fw`

```
#!/bin/bash
# Default IPv6 Policies

#Flush current policies
ip6tables -F

# Set default chain policies
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT DROP

# Allow established sessions to receive traffic
ip6tables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Accept on localhost
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A OUTPUT -o lo -j ACCEPT

# Save Changes
service ip6tables save

# Service
systemctl restart ip6tables
systemctl status ip6tables
```

Results

```
[root@myhost~]# ip6tables -L -nv
Chain INPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all      *      *       ::/0                 ::/0                 ctstate RELATED,ESTABLISHED
    0     0 ACCEPT     all      lo     *       ::/0                 ::/0        

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all      *      lo      ::/0                 ::/0        

```

## SELinux

As much as I love the idea of SELinux, the reality is that developers as a whole have not adopted its use. As a consequence, I have lost far too many hours trouble shooting installation failures to identify SELinux as the culprit. I have not given up entirely on SELinux, but I would advise setting it to "permissive" when installing new services and testing. Once testing is complete, set SELinux to "enforcing" and test again.

Update the SELinux config file `vi /etc/selinux/config` to `SELINUX=permissive`. Execute `setenforce 0` to set the current session to permissive or reboot to utilize the updated config.

## Time zone

After installation, the default time zone is America/New_York. CentOS 7 uses `timedatectl` to manage time and date related settings.

Check current settings using `timedatectl`

```
[root@myhost ~]# timedatectl
      Local time: Fri 2016-04-01 18:01:44 EDT
  Universal time: Fri 2016-04-01 22:01:44 UTC
        RTC time: Fri 2016-04-01 22:01:43
       Time zone: America/New_York (EDT, -0400)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2016-03-13 01:59:59 EST
                  Sun 2016-03-13 03:00:00 EDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2016-11-06 01:59:59 EDT
                  Sun 2016-11-06 01:00:00 EST
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
      Local time: Fri 2016-04-01 15:08:10 PDT
  Universal time: Fri 2016-04-01 22:08:10 UTC
        RTC time: Fri 2016-04-01 22:08:10
       Time zone: America/Los_Angeles (PDT, -0700)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2016-03-13 01:59:59 PST
                  Sun 2016-03-13 03:00:00 PDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2016-11-06 01:59:59 PDT
                  Sun 2016-11-06 01:00:00 PST
```

References http://www.server-world.info/en/note?os=CentOS_7&p=timezone

## Time & Date

View the current date and time using `date`.

Set the current local time and date using `timedatectl 2016-04-02 17:48:12`. The result is `Sat Apr  2 17:48:12 PDT 2016`.

## Network Time
Time synchronization can play a big role in kerberos authentication and other services. By default "chrony" is installed instead of the "ntpd." To update chrony time sources, `# vi /etc/chrony.conf` and update or add "server" values.

## EPEL (optional)
To install Extra Packages for Linux (EPEL), simply install the package from the CentOS base repository.

```
[root@myhost~]# yum install epel-release
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

If using CentOS 7 as an oVirt Guest (virtual machine), install your version of the oVirt repository and guest agent. I am using oVirt 4.0.

### oVirt Repository

```
[root@myhost ~]# yum install -y http://resources.ovirt.org/pub/yum-repo/ovirt-release40.rpm
```

Results

```
================================================================================
 Package              Arch        Version           Repository             Size
================================================================================
Installing:
 ovirt-release40      noarch      4.0.6.1-1         /ovirt-release40      6.0 k

Transaction Summary
================================================================================
Install  1 Package

Total size: 6.0 k
Installed size: 6.0 k
Is this ok [y/d/N]:
```

### oVirt Guest

```
[root@myhost ~]# yum install ovirt-guest-agent-common
```

Results

```
================================================================================
 Package                       Arch        Version              Repository
                                                                           Size
================================================================================
Installing:
 ovirt-guest-agent-common      noarch      1.0.12-4.el7         epel       69 k
Installing for dependencies:
 libnl                         x86_64      1.1.4-3.el7          base      128 k
 python-ethtool                x86_64      0.8-5.el7            base       33 k
 qemu-guest-agent              x86_64      10:2.5.0-3.el7       base      133 k
 usermode                      x86_64      1.111-5.el7          base      193 k

Transaction Summary
================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 556 k
Installed size: 1.8 M
Is this ok [y/d/N]:
```

Enable and start the agent.

```
[root@myhost ~]# systemctl enable ovirt-guest-agent
[root@myhost ~]# systemctl status ovirt-guest-agent
● ovirt-guest-agent.service - oVirt Guest Agent
   Loaded: loaded (/usr/lib/systemd/system/ovirt-guest-agent.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
[root@myhost ~]# systemctl start ovirt-guest-agent
[root@myhost ~]# systemctl status ovirt-guest-agent
● ovirt-guest-agent.service - oVirt Guest Agent
   Loaded: loaded (/usr/lib/systemd/system/ovirt-guest-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2017-02-18 15:47:56 PST; 4s ago
  Process: 1388 ExecStartPre=/bin/chown ovirtagent:ovirtagent /run/ovirt-guest-agent.pid (code=exited, status=0/SUCCESS)
  Process: 1385 ExecStartPre=/bin/touch /run/ovirt-guest-agent.pid (code=exited, status=0/SUCCESS)
  Process: 1383 ExecStartPre=/sbin/modprobe virtio_console (code=exited, status=0/SUCCESS)
 Main PID: 1391 (python)
   CGroup: /system.slice/ovirt-guest-agent.service
           └─1391 /usr/bin/python /usr/share/ovirt-guest-agent/ovirt-guest-ag...

Feb 18 15:47:56 myhost.mydomain.net systemd[1]: Starting oVirt Guest Agent...
Feb 18 15:47:56 myhost.mydomain.net systemd[1]: Started oVirt Guest Agent.
Feb 18 15:47:56 myhost.mydomain.net userhelper[1401]: pam_succeed_if(ovirt-...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1400]: pam_succeed_if(ovirt-...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1401]: running '/usr/share/o...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1400]: running '/usr/share/o...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1403]: pam_succeed_if(diskma...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1403]: running '/usr/share/o...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1411]: pam_succeed_if(ovirt-...
Feb 18 15:47:56 myhost.mydomain.net userhelper[1411]: running '/usr/share/o...
Hint: Some lines were ellipsized, use -l to show in full.
```

## CloudInit (optional)

CloudInit handles early initialization of virtual machines. I use the cloud-init service with oVirt to configure network settings, passwords, and other settings when initializing from virtual machines templates.

```
[root@myhost~]# yum install cloud-init
```

Results

```
================================================================================
 Package                          Arch   Version                  Repository
                                                                           Size
================================================================================
Installing:
 cloud-init                       x86_64 0.7.5-10.el7.centos.1    extras  418 k
Installing for dependencies:
 PyYAML                           x86_64 3.10-11.el7              base    153 k
 audit-libs-python                x86_64 2.6.5-3.el7              base     70 k
 checkpolicy                      x86_64 2.5-4.el7                base    290 k
 jbigkit-libs                     x86_64 2.0-11.el7               base     46 k
 libcgroup                        x86_64 0.41-11.el7              base     65 k
 libjpeg-turbo                    x86_64 1.2.90-5.el7             base    134 k
 libsemanage-python               x86_64 2.5-5.1.el7_3            updates 104 k
 libtiff                          x86_64 4.0.3-27.el7_3           updates 170 k
 libwebp                          x86_64 0.3.0-3.el7              base    170 k
 libyaml                          x86_64 0.1.4-11.el7_0           base     55 k
 net-tools                        x86_64 2.0-0.17.20131004git.el7 base    304 k
 policycoreutils-python           x86_64 2.5-11.el7_3             updates 445 k
 python-IPy                       noarch 0.75-6.el7               base     32 k
 python-backports                 x86_64 1.0-8.el7                base    5.8 k
 python-backports-ssl_match_hostname
                                  noarch 3.4.0.2-4.el7            base     12 k
 python-chardet                   noarch 2.2.1-1.el7_1            base    227 k
 python-cheetah                   x86_64 2.4.4-5.el7.centos       extras  341 k
 python-jsonpatch                 noarch 1.2-3.el7.centos         extras   14 k
 python-jsonpointer               noarch 1.9-2.el7                base     13 k
 python-markdown                  noarch 2.4.1-1.el7.centos       extras  186 k
 python-pillow                    x86_64 2.0.0-19.gitd1c6db8.el7  base    438 k
 python-prettytable               noarch 0.7.2-2.el7.centos       extras   37 k
 python-pygments                  noarch 1.4-9.el7                base    599 k
 python-requests                  noarch 2.6.0-1.el7_1            base     94 k
 python-setuptools                noarch 0.9.8-4.el7              base    396 k
 python-six                       noarch 1.9.0-2.el7              base     29 k
 python-urllib3                   noarch 1.10.2-2.el7_1           base    100 k
 python2-boto                     noarch 2.44.0-1.el7             epel    1.7 M
 python2-pyasn1                   noarch 0.1.9-7.el7              base    100 k
 python2-rsa                      noarch 3.4.1-1.el7              epel     67 k
 setools-libs                     x86_64 3.3.8-1.1.el7            base    612 k

Transaction Summary
================================================================================
Install  1 Package (+31 Dependent packages)

Total download size: 7.3 M
Installed size: 31 M
Is this ok [y/d/N]:
```

After installation, the cloud-init service is already enabled and will be running after reboot. I find its constant complaining to be a nuisance, so I stop and disable the service until it is time to seal the virtual machine.

```
[root@myhost~]# systemctl disable cloud-init
```

Reference

* https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.0/html/virtual_machine_management_guide/sect-using_cloud-init_to_automate_the_configuration_of_virtual_machines

## Spacewalk 2.6 Client (optional)

Spacewalk is the upstream project for Satellite 5 aka Satellite Classic. I use it for patch and configuration management and a guide to build the Spacewalk 2.6 Server can be found at [Spacewalk 2.6](https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Configuration-and-Patch-Management-with-Spacewalk-2.6-on-CentOS-7.3.1611-Minimal).

### Spacewalk Client Repository

Install the repository's package matching the Spacewalk server.

```
[root@myclient ~]# yum install http://yum.spacewalkproject.org/2.6-client/RHEL/7/x86_64/spacewalk-client-repo-2.6-0.el7.noarch.rpm
```

Results

```
================================================================================
 Package         Arch   Version   Repository                               Size
================================================================================
Installing:
 spacewalk-client-repo
                 noarch 2.6-0.el7 /spacewalk-client-repo-2.6-0.el7.noarch 426

Transaction Summary
================================================================================
Install  1 Package

Total size: 426
Installed size: 426
Is this ok [y/d/N]:
```

### Spacewalk Client Packages

Spacewalk client packages to register and utilize core Spacewalk features like registration.

```
[root@myhost~]#  yum install m2crypto rhn-check rhn-client-tools rhn-setup rhnsd yum-rhn-plugin
```

Results

```
================================================================================
 Package              Arch       Version             Repository            Size
================================================================================
Installing:
 m2crypto             x86_64     0.21.1-17.el7       base                 429 k
 rhn-check            noarch     2.6.8-1.el7         spacewalk-client      58 k
 rhn-client-tools     noarch     2.6.8-1.el7         spacewalk-client     483 k
 rhn-setup            noarch     2.6.8-1.el7         spacewalk-client      94 k
 rhnsd                x86_64     5.0.25-1.el7        spacewalk-client      47 k
 yum-rhn-plugin       noarch     2.6.3-1.el7         spacewalk-client      84 k
Installing for dependencies:
 libxml2-python       x86_64     2.9.1-6.el7_2.3     base                 247 k
 pyOpenSSL            x86_64     0.13.1-3.el7        base                 133 k
 pygobject2           x86_64     2.28.6-11.el7       base                 226 k
 python-dmidecode     x86_64     3.10.13-11.el7      base                  82 k
 python-gudev         x86_64     147.2-7.el7         base                  18 k
 python-hwdata        noarch     1.7.3-4.el7         base                  32 k
 rhnlib               noarch     2.6.3-1.el7         spacewalk-client      68 k

Transaction Summary
================================================================================
Install  6 Packages (+7 Dependent packages)

Total download size: 2.0 M
Installed size: 8.3 M
Is this ok [y/d/N]:
```

To utilize Spacewalk, the host must be registered and additional functionality will require additional packages and configuration. Details on the steps to complete a Spacewalk client setup are found at the URL below.


* https://github.com/rharmonson/richtech/wiki/OSVDC-Series:-Configuration-and-Patch-Management-with-Spacewalk-2.6-on-CentOS-7.3.1611-Minimal#spacewalk-client


## Update

If using the CentOS 7 Minimal installation media, update prior to building services. If using the CentOS 7 NetInstall media, there should be no updates needed.

```
[root@myhost ~]# yum update
```

## VM Template

If using the Linux host created using the instructions above as a virtual machine (VM) template, I use the following process to prepare the VM.

1. Clean yum
1. Clear machine-id
1. Enable cloud-init
1. Delete SSH host keys
1. Delete history and logs
1. sys-unconfig
1. Convert virtual machine to template
1. Deploy virtual machine using template

To simplify the process further, create a file, set it as executable and paste the following. Execute using `./sealvm.sh` as the last step in the template build process.

```
[root@myhost~]# touch sealvm.sh
[root@myhost~]# chmod +x sealvm.sh
[root@myhost~]# vi sealvm.sh
```

cut+paste

```
#!/bin/bash
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

Reference

* https://access.redhat.com/documentation/en-us/red_hat_virtualization/4.0/html/virtual_machine_management_guide/chap-templates#sect-Sealing_Virtual_Machines_in_Preparation_for_Deployment_as_Templates

## Done!?

The build is complete. However, you may want to consider the following:

## Disable Postfix

Depending on the purpose of the system, postfix may not be needed.

```
[root@myhost ~]# systemctl stop postfix
[root@myhost ~]# systemctl disable postfix
Removed symlink /etc/systemd/system/multi-user.target.wants/postfix.service.
[root@myhost ~]# yum remove postfix
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
Is this ok [y/N]:
```

## Additional Packages

I install a number of optional packages for my builds including:

```
[root@myhost ~]# yum install deltarpm yum-utils tmux
```

Results

```
================================================================================
 Package            Arch            Version                 Repository     Size
================================================================================
Installing:
 deltarpm           x86_64          3.6-3.el7               base           82 k
 tmux               x86_64          1.8-4.el7               base          243 k
 yum-utils          noarch          1.1.31-40.el7           base          116 k

Transaction Summary
================================================================================
Install  3 Packages

Total download size: 441 k
Installed size: 1.1 M
Is this ok [y/d/N]:
```
{% include links.html %}
