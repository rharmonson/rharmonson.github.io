---
title: CentOS 7 Installation Guide on Raspberry PI
published: true
tags: [linux, centos, hardware, sysadmin]
keywords: firewalld, NetworkManager, iptables, raspberry
last_updated: April 2, 2017
summary: "The purpose of this article is to describe how to install CentOS 7 on the Raspberry PI 3 B."
sidebar: cos7instpi_sidebar
permalink: cos7instpi.html
folder: 2017
toc: false
commentIssueId: 4
---

## Purpose

The purpose of this article is to describe how to install CentOS 7 on the Raspberry PI 3 B for use as a starting point to install light-weight services such as DNS, NTP, DHCP, Apache, etc. This guide has been tested successfully using CentOS 7.2.1511 and 7.3.1611.

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
The default root password is <b>centos</b>."
%}

### Media

Begin, by obtaining the CentOS media from:

http://mirror.centos.org/altarch/7/isos/armhfp

I will be using:

http://mirror.centos.org/altarch/7/isos/armhfp/CentOS-Userland-7-armv7hl-Minimal-1603-RaspberryPi3.img.xz

### Create

There are number of methods, and I found this website on xmodulo.com to be very good.

http://xmodulo.com/write-raspberry-pi-image-sd-card.html

## README

There is a `/README` file that describes remaining steps to complete the Raspbery PI 3 setup including how to expand the root (/) partition to capacity of the media. I found `touch ./rootfs-repartition` failed on reboot with a kernel panic with SanDisk and Samsung 64 GB media, but it did worked with smaller SanDisk 8 GB media.

Follow the instructions to expand the root filesystem and install the wireless drivers. If you receive a kernel panic on reboot after resizing the root partition, see the section below titled "Manual Expand RootFS."

```
[root@centos-rpi3 ~]# cat /root/README
== CentOS 7 userland ==

If you want to automatically resize your / partition, just type the following (as root user):
touch /.rootfs-repartition
systemctl reboot

For wifi on the rpi3, just proceed with those steps :

curl --location https://github.com/RPi-Distro/firmware-nonfree/raw/54bab3d6a6d43239c71d26464e6e10e5067ffea7/brcm80211/brcm/brcmfmac43430-sdio.bin > /usr/lib/firmware/brcm/brcmfmac43430-sdio.bin

curl --location https://github.com/RPi-Distro/firmware-nonfree/raw/54bab3d6a6d43239c71d26464e6e10e5067ffea7/brcm80211/brcm/brcmfmac43430-sdio.txt > /usr/lib/firmware/brcm/brcmfmac43430-sdio.txt

systemctl reboot
```

## Manual Expand RootFS

There is a `/README` file that describes remaining steps to complete the Raspbery PI 3 setup including how to expand the root (/) partition to capacity of the media. I tried it and it resulted in a kernel panic.

Below is the method I used to expand the root filesystem.

```
[root@centos-rpi3 ~]# fdisk -l

Disk /dev/mmcblk0: 63.9 GB, 63864569856 bytes, 124735488 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a4da3

        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk0p1            2048      616447      307200    c  W95 FAT32 (LBA)
/dev/mmcblk0p2          616448     1665023      524288   82  Linux swap / Solaris
/dev/mmcblk0p3         1665024     5859327     2097152   83  Linux
[root@centos-rpi3 ~]# fdisk /dev/mmcblk0
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p

Disk /dev/mmcblk0: 63.9 GB, 63864569856 bytes, 124735488 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000a4da3

        Device Boot      Start         End      Blocks   Id  System
/dev/mmcblk0p1            2048      616447      307200    c  W95 FAT32 (LBA)
/dev/mmcblk0p2          616448     1665023      524288   82  Linux swap / Solaris
/dev/mmcblk0p3         1665024     5859327     2097152   83  Linux

Command (m for help): d
Partition number (1-3, default 3): 3
Partition 3 is deleted

Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
Partition number (3,4, default 3):
First sector (1665024-124735487, default 1665024):
Using default value 1665024
Last sector, +sectors or +size{K,M,G} (1665024-124735487, default 124735487):
Using default value 124735487
Partition 3 of type Linux and of size 58.7 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
[root@centos-rpi3 ~]# reboot
```

After reboot, resize the file system using resize2fs.

```
[root@centos-rpi3 ~]# resize2fs /dev/mmcblk0p3
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/mmcblk0p3 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 8
The filesystem on /dev/mmcblk0p3 is now 15422208 blocks long.
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

I have no use for the wireless adapter nor bluetooth. Disabling the devices will increase security and reduce electrical / heat.

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

Since I have no intention of using wireless, disable the wpa_suplicant service using `systemctl disable wpa_supplicant.service`.

Reference: http://raspberrypi.stackexchange.com/questions/43720/disable-wifi-wlan0-on-pi-3#43721

## NetworkManager & firewalld

I am not a fan of NetworkManager nor firewalld on CentOS Minimal installations, so they have to go!

```
[root@centos-rpi3 ~]# systemctl stop NetworkManager firewalld
[root@centos-rpi3 ~]# systemctl disable NetworkManager firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/basic.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
```

`[root@centos-rpi3 ~]# yum remove NetworkManager NetworkManager-libnm firewalld`

Results

```
Dependencies Resolved

================================================================================
 Package                 Arch       Version           Repository           Size
================================================================================
Removing:
 NetworkManager          armv7hl    1:1.0.6-27.el7    @centos-base_rbf    8.7 M
 NetworkManager-libnm    armv7hl    1:1.0.6-27.el7    @centos-base_rbf    1.2 M
 firewalld               noarch     0.3.9-14.el7      @centos-base_rbf    2.3 M
Removing for dependencies:
 NetworkManager-team     armv7hl    1:1.0.6-27.el7    @centos-base_rbf     31 k
 NetworkManager-tui      armv7hl    1:1.0.6-27.el7    @centos-base_rbf    209 k
 NetworkManager-wifi     armv7hl    1:1.0.6-27.el7    @centos-base_rbf    105 k

Transaction Summary
================================================================================
Remove  3 Packages (+3 Dependent packages)

Installed size: 13 M
Is this ok [y/N]:
```

```
[root@centos-rpi3 ~]# yum install iptables-services
Loaded plugins: fastestmirror
base                                                     | 3.6 kB     00:00
extras                                                   | 2.9 kB     00:00
updates                                                  | 2.9 kB     00:00
(1/4): extras/7/armhfp/primary_db                          |  14 kB   00:00
(2/4): base/7/armhfp/group_gz                              | 154 kB   00:00
(3/4): updates/7/armhfp/primary_db                         | 1.0 MB   00:01
(4/4): base/7/armhfp/primary_db                            | 2.6 MB   00:03
Determining fastest mirrors
Resolving Dependencies
--> Running transaction check
---> Package iptables-services.armv7hl 0:1.4.21-16.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package                  Arch           Version              Repository   Size
================================================================================
Installing:
 iptables-services        armv7hl        1.4.21-16.el7        base         49 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 49 k
Installed size: 24 k
Is this ok [y/d/N]: y
Downloading packages:
warning: /var/cache/yum/armhfp/7/base/packages/iptables-services-1.4.21-16.el7.armv7hl.rpm: Header V4 RSA/SHA1 Signature, key ID 62505fe6: NOKEY
Public key for iptables-services-1.4.21-16.el7.armv7hl.rpm is not installed
iptables-services-1.4.21-16.el7.armv7hl.rpm                |  49 kB   00:00
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-userland-release-7-2.1511.el7.centos.0.4.armv7hl (@centos-base_rbf)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Is this ok [y/N]: y
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
Importing GPG key 0x62505FE6:
 Userid     : "CentOS AltArch SIG - Arm32 (https://wiki.centos.org/SpecialInterestGroup/AltArch/Arm32) <security@centos.org>"
 Fingerprint: 4d9e 39f1 499c a21d d289 77f8 cafe f11b 6250 5fe6
 Package    : centos-userland-release-7-2.1511.el7.centos.0.4.armv7hl (@centos-base_rbf)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : iptables-services-1.4.21-16.el7.armv7hl                      1/1
  Verifying  : iptables-services-1.4.21-16.el7.armv7hl                      1/1

Installed:
  iptables-services.armv7hl 0:1.4.21-16.el7

Complete!
[root@centos-rpi3 ~]# systemctl enable iptables
Created symlink from /etc/systemd/system/basic.target.wants/iptables.service to /usr/lib/systemd/system/iptables.service.
[root@centos-rpi3 ~]# systemctl enable network
network.service is not a native service, redirecting to /sbin/chkconfig.
Executing /sbin/chkconfig network on
```

If you reboot at this point without completing the next step, execute `dhclient` to obtain an IP address.

## Network Interface

I use static IP addresses for infrastructure related services. As such, we need to update ifcfg-eth0.

`# vi /etc/sysconfig/network-scripts/ifcfg-eth0`

Update to reflect your IP topology:

```
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.253
NETMASK=255.255.255.0
GATEWAY=192.168.1.254
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
            Kernel: Linux 4.1.19-v7
      Architecture: arm
```

After `reboot` verify changes using `ping -n 5 www.google.com`.

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
      Local time: Mon 2016-10-24 19:44:45 PDT
  Universal time: Tue 2016-10-25 02:44:45 UTC
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

`yum install yum-utils deltarpm tmux`.

Results

```
================================================================================
 Package               Arch           Version                 Repository   Size
================================================================================
Installing:
 deltarpm              armv7hl        3.6-3.el7               base         81 k
 tmux                  armv7hl        1.8-4.el7               base        211 k
 yum-utils             noarch         1.1.31-34.el7           base        113 k
Installing for dependencies:
 libevent              armv7hl        2.0.21-4.el7            base        189 k
 python-chardet        noarch         2.2.1-1.el7_1           base        227 k
 python-kitchen        noarch         1.1.1-5.el7             base        267 k

Transaction Summary
================================================================================
Install  3 Packages (+3 Dependent packages)

Total download size: 1.1 M
Installed size: 4.1 M
Is this ok [y/d/N]: y
```

Update the system. `tmux` is an alternative to `screen`. It allows reconnecting to a tmux session with ease using `tmux attach`.

```
[root@myhost ~]# tmux new -s update
[root@myhost ~]# yum update -y && reboot
```

## Done!?

At this point, you are ready to add repositories and install packages.

Please star to let me know you found this article useful or open an issue with questions or comments.

Have fun!

{% include links.html %}
