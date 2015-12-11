---
date: '2015-06-06'
tags:
- dnf
- yum
- fedora
- centos
- linux
title: "Yum/DNF Keep Old Packages"
url: "linux/2015/06/06/yum-dnf-keep-old-packages"
---

When you upgrade packages on RedHat based systems, the newer package replaces the older one except for _install only_ packages e.g. kernel packages.
Upgrading kernel package(s) with yum, dnf and even apt leaves 3 older versions—by default—of the kernel package(s) on the system. This can be useful in cases where you need to use to an older version.
<!--more-->

Once you have more than 3 versions, yum & dnf will clean the older versions of the _install only_ package(s). To retain more than 3 versions, set `installonly_limit`  in `/etc/yum.conf` or `/etc/dnf/dnf.conf` to a number that you want, e.g. keep 5 versions:
```ini
installonly_limit=5
```

To keep all older kernel packages forever, set `installonly_limit` to **0** i.e. never remove older versions of _install only_ packages.
```ini
installonly_limit=0
```

