---
title: CentOS 7 Installation Guide on Raspberry PI
published: true
tags: [linux, centos, hardware, sysadmin]
keywords: firewalld, NetworkManager, iptables, raspberry
last_updated: November 25, 2017
summary: "The purpose of this article is to describe how to install CentOS 7 on the Raspberry PI 3 B."
sidebar: cos7instpi_sidebar
permalink: cos7instpi.html
folder: 2017
toc: false
commentIssueId: 4
---

## Purpose

The purpose of this article is to describe how to install CentOS 7 on the Raspberry PI 3 B for use as a starting point to install light-weight services such as DNS, NTP, DHCP, Apache, etc. This guide has been tested using CentOS 7.4.1708.

## Raspberry PI 3

At the cost of $35 and an additional $20 to $30 in accessories the PI3 is a steal for a server providing reasonably light-weight services or workloads.

PI3 Specifications

*    A 1.2GHz 64-bit quad-core ARMv8 CPU
*    802.11n Wireless LAN
*    Bluetooth 4.1
*    Bluetooth Low Energy (BLE)
*    1GB RAM
*    4 USB ports
*    40 GPIO pins
*    Full HDMI port
*    Ethernet port
*    Combined 3.5mm audio jack and composite video
*    Camera interface (CSI)
*    Display interface (DSI)
*    Micro SD card slot (now push-pull rather than push-push)
*    VideoCore IV 3D graphics core

## Install CentOS 7

### General information

https://wiki.centos.org/SpecialInterestGroup/AltArch/Arm32/RaspberryPi3

{% include tip.html content="Root Password<br/>
<br/>
The default root password is <b>centos</b>."
%}

### Media

Begin, by obtaining the CentOS media from:

http://mirror.centos.org/altarch/7/isos/armhfp

I will be using:

http://mirror.centos.org/altarch/7/isos/armhfp/CentOS-Userland-7-armv7hl-Minimal-1708-RaspberryPi3.img.xz

### Create

There are number of methods, and I found this website on xmodulo.com to be very good.

http://xmodulo.com/write-raspberry-pi-image-sd-card.html

## README

There is a `/README` file that describes remaining steps to complete the Raspbery PI 3 setup including how to expand the root (/) partition to capacity of the media. Follow the instructions to expand the root filesystem using `/usr/bin/rootfs-expand`.

```
== CentOS 7 userland ==

If you want to automatically resize your / partition, just type the following (as root user):
/usr/bin/rootfs-expand
```

## Services

Immediately after installation, execute `systemctl` and note the two failing services network and kdump. To fix, I did the following:

### network.service

Executing `systemctl start network` resulted with an error like "Failed to start LSB: Bring up/down networking." Not terribly helpful. The solution was pretty simple, however. Execute `echo "NETWORKING=yes" > /etc/sysconfig/network` for the "network" file is absent.

### kdump.service

Executing `systemctl start kdump` then `journalctl -xe` shows the message "Kdump not supported on this kernel." Disable the service by executing `systemctl disable kdump`.

### systemd-tmpfiles-setup.service

Periodically, I saw an error with systemd-tmpfiles-setup.service. It would come and go, so I ignored it. Further research is needed.

### Disable Wifi/BT

I have no use for the wireless adapter nor bluetooth. Disabling the devices will increase security and reduce electrical / heat if not significantly.

```
[root@centos-rpi3 ~]# vi /etc/modprobe.d/raspi-blklst.conf
```

Add

```
#wifi
blacklist brcmfmac
blacklist brcmutil

#bt
blacklist btbcm
blacklist hci_uart
```

Since I have no intention of using wireless, disable the wpa_suplicant service using `systemctl disable wpa_supplicant.service`. Using 7.4.1708 this service was not enabled.

Reference: http://raspberrypi.stackexchange.com/questions/43720/disable-wifi-wlan0-on-pi-3#43721

## NetworkManager & firewalld

I am not a fan of NetworkManager nor firewalld on CentOS Minimal installations, so they have to go! However, we will need `iptables-services` to manage iptables in the absence of firewalld.

```
[root@centos-rpi3 ~]# systemctl stop NetworkManager firewalld
[root@centos-rpi3 ~]# systemctl disable NetworkManager firewalld
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
```

`[root@centos-rpi3 ~]# yum remove NetworkManager NetworkManager-libnm firewalld`

Results

```
Dependencies Resolved

================================================================================
 Package                 Arch       Version           Repository           Size
================================================================================
Removing:
 NetworkManager          armv7hl    1:1.8.0-11.el7    @updates            4.3 M
 NetworkManager-libnm    armv7hl    1:1.8.0-11.el7    @updates            5.4 M
 firewalld               noarch     0.4.4.4-6.el7     @centos-base_rbf    1.8 M
Removing for dependencies:
 NetworkManager-team     armv7hl    1:1.8.0-11.el7    @updates             44 k
 NetworkManager-tui      armv7hl    1:1.8.0-11.el7    @updates            199 k
 NetworkManager-wifi     armv7hl    1:1.8.0-11.el7    @updates            122 k

Transaction Summary
================================================================================
Remove  3 Packages (+3 Dependent packages)

Installed size: 12 M
Is this ok [y/N]:
```

Install `iptables-services`

```
[root@centos-rpi3 ~]# yum install iptables-services
Loaded plugins: fastestmirror
base                                                     | 3.6 kB     00:00
centos-kernel                                            | 2.9 kB     00:00
extras                                                   | 2.9 kB     00:00
ovirt-4.1                                                | 3.0 kB     00:00
updates                                                  | 2.9 kB     00:00
Loading mirror speeds from cached hostfile
 * ovirt-4.1: resources.ovirt.org
Resolving Dependencies
--> Running transaction check
---> Package iptables-services.armv7hl 0:1.4.21-18.0.1.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                Arch         Version                 Repository    Size
================================================================================
Installing:
 iptables-services      armv7hl      1.4.21-18.0.1.el7       updates       50 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 50 k
Installed size: 25 k
Is this ok [y/d/N]:
```

Enable iptables-services.

```
[root@centos-rpi3 ~]# systemctl enable iptables ip6tables
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
Created symlink from /etc/systemd/system/basic.target.wants/ip6tables.service to /usr/lib/systemd/system/ip6tables.service.
[root@centos-rpi3 ~]# systemctl start iptables ip6tables
```

If you reboot at this point without completing the next step, execute `dhclient` to obtain an IP address.

## Network Interface

I use static IP addresses for infrastructure related services. As such, we need to update ifcfg-eth0.

```
[root@centos-rpi3 ~]# vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

Update to reflect your IP address topology:

```
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.101
NETMASK=255.255.255.0
GATEWAY=192.168.1.254
```

Reboot or restart the network service for changes to take effect.

```
[root@centos-rpi3 ~]# systemctl restart network
```

## Name Resolution

`# vi /etc/resolv.conf`

```
search myhost.mydomain.net
nameserver 64.6.64.6 # Verisign: Reston, Virginia
nameserver 64.6.65.6 # Verisign: Reston, Virginia
```

## Host Name

Use hostnamectl to set host name.

```
[root@centos-rpi3 ~]# hostnamectl set-hostname myhost.mydomain.net
[root@centos-rpi3 ~]# hostnamectl
   Static hostname: myhost.mydomain.net
         Icon name: computer
        Machine ID: c86851c595a149019a820550c3ccec08
           Boot ID: 9d27034c55c84cbda2d7dcf8f7f229e2
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 4.9.50-v7.1.el7
      Architecture: arm
```

Restarting the `network` service will not suffice, so `reboot` to verify changes using `ping -c3 www.google.com`.

Results

```
[root@myhost ~]# ping -c3 www.google.com
PING www.google.com (172.217.5.100) 56(84) bytes of data.
64 bytes from sfo03s07-in-f4.1e100.net (172.217.5.100): icmp_seq=1 ttl=51 time=26.6 ms
64 bytes from sfo03s07-in-f4.1e100.net (172.217.5.100): icmp_seq=2 ttl=51 time=26.7 ms
64 bytes from sfo03s07-in-f4.1e100.net (172.217.5.100): icmp_seq=3 ttl=51 time=26.5 ms

--- www.google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 26.594/26.682/26.765/0.201 ms
```

## Time & Date

Use timedatectl to set time, timezone, and/or date.

```
[root@myhost ~]# timedatectl list-timezones | grep -i angeles
America/Los_Angeles
[root@myhost ~]# timedatectl set-timezone America/Los_Angeles
[root@myhost ~]# timedatectl
      Local time: Mon 2017-11-24 19:44:45 PDT
  Universal time: Tue 2017-11-25 02:44:45 UTC
        RTC time: n/a
       Time zone: America/Los_Angeles (PDT, -0700)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: yes
 Last DST change: DST began at
                  Sun 2016-03-13 01:59:59 PST
                  Sun 2016-03-13 03:00:00 PDT
 Next DST change: DST ends (the clock jumps one hour backwards) at
                  Sun 2016-11-06 01:59:59 PDT
                  Sun 2016-11-06 01:00:00 PST
```

## yum

Check for yum update using `yum update yum`, then install preferred packages. For example, I install yum related packages and tmux.

```
[root@myhost ~]# yum install yum-utils deltarpm tmux
```

Results

```
================================================================================
 Package               Arch           Version                 Repository   Size
================================================================================
Installing:
 deltarpm              armv7hl        3.6-3.el7               base         81 k
 tmux                  armv7hl        1.8-4.el7               base        211 k
 yum-utils             noarch         1.1.31-42.el7           base        117 k
Installing for dependencies:
 libevent              armv7hl        2.0.21-4.el7            base        189 k
 libxml2-python        armv7hl        2.9.1-6.el7.3           base        233 k

Transaction Summary
================================================================================
Install  3 Packages (+2 Dependent packages)

Total download size: 832 k
Installed size: 2.9 M
Is this ok [y/d/N]:
```

Update the system. `tmux` is an alternative to `screen`. It allows reconnecting to a tmux session with ease when using SSH using `tmux attach`.

```
[root@myhost ~]# tmux new -s update
[root@myhost ~]# yum update -y && reboot
```

## Done!?

At this point, you are ready to add repositories and install packages.

Please star to let me know you found this article useful or open an issue with questions or comments.

Have fun!

{% include links.html %}
