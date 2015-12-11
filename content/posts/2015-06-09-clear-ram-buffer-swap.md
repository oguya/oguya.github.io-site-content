---
date: '2015-06-09'
tags:
- cache
- swap
- linux
title: "Clear RAM Memory Cache, Buffer & Swap Space"
url: "linux/2015/06/09/clear-ram-buffer-swap"
---

GNU Linux has implemented efficient memory management algorithms & tools which gives you the power & flexibility to control both the physical & virtual memory on your system.
<!--more-->

### Clear Cache

There are 3 options to clear cache without interrupting any processes or services:

- Clear page cache:
```sh
# echo 1 > /proc/sys/vm/drop_cache
```

- Clear [dentries](http://unix.stackexchange.com/a/4403) and [inodes](http://unix.stackexchange.com/a/4403)
```sh
# echo 2 > /proc/sys/vm/drop_cache
```

- Clear PageCache, dentries and inodes
```sh
# echo 3 > /proc/sys/vm/drop_cache
```

### Clear Swap Space

- Clearing swap space is as easy as disabling & re-enabling the devices/files used for swaping.
```sh
# swapoff -a && swapon -a
```

### Further Reading

1. [What is a Superblock, Inode, Dentry and a File?](https://unix.stackexchange.com/questions/4402/what-is-a-superblock-inode-dentry-and-a-file/4403#4403)
