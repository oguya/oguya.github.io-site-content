---
date: '2015-12-02'
tags:
- fedora
- linux
title: "Upgrading Fedora 22 to 23 using DNF"
---
This is a short & simple guide for upgrading Fedora 22 to Fedora 23 using DNF
<!--more-->

Live Fedora upgrades using [dnf](https://fedoraproject.org/wiki/Dnf) or [yum](https://fedoraproject.org/wiki/Yum) usually works well with typical installations but that doesn't mean you won't experience errors during the process.
In this guide, I'll break the entire process into 3 stages:

- Pre-upgrade
- Upgrade
- Post-upgrade


## Pre-upgrade

- **Backup your Data:**</br>
Backup all your important data before doing anything. It's always a good idea to have backups which you can fallback to incase something goes utterly wrong the upgrade.

- **Run updates & reboot:**</br>
Upgrade your Fedora 22 packages & reboot before you can start the upgrade, just to make sure you're running the latest kernel & other packages.
```sh
# dnf upgrade
# systemctl reboot
```

## Upgrade

- Import Fedora 23 GPG Key, clean dnf local cache & finally start the upgrade.
```sh
# rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-23-$(uname -i)
# dnf upgrade
# dnf clean all
# dnf --releasever=23 --setopt=deltarpm=false distro-sync
```

## Post-Upgrade

- After the upgrade, invoke SELinux to autorelabel all files/dirs and finally reboot. Autorelabling will occur during the next boot process.
```sh
# touch /.autorelabel
# systemctl reboot
```

## Gotchas!!!

- During the upgrade I bumped into a couple of errors related to missing dependencies.
```
# dnf --releasever=23 --setopt=deltarpm=false distro-sync
Last metadata expiration check performed 0:27:00 ago on Wed Dec  2 10:19:02 2015.
Error: package rubygem-celluloid-0.15.2-2.fc22.noarch requires rubygem(timers) < 1.2, but none of the providers can be installed
package libxml2-python-2.9.3-1.fc22.x86_64 requires libxml2 = 2.9.3-1.fc22, but none of the providers can be installed
(try to add '--allowerasing' to command line to replace conflicting packages)
```
- Since both package dependencies were available on Fedora 23 repo, I switched to that repo & upgraded them before running the upgrade process again. I think this is not the recommended way of fixing such issues, but it worked!
```sh
# dnf --releasever=23 install --best --allowerasing rubygem-timers libxml2
```

- Another thing to note is that DNF upgrades are heavily dependent on your network. So it's always a good idea to use local mirrors to speed up the upgrade process.
