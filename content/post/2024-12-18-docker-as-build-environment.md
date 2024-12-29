+++
date = '2024-12-18T17:00:00+01:00'
draft = false
title = 'Docker as Build Environment'
+++

You have a small/mid sized server and you want to install new software or custom kernel on it.
But it take forever to build anything on it compared to your brand new 8 or 12 core modern laptop with fast NVMe SSD.

In addition, you don´t want to pollute your server with all the build dependencies.

Docker is here to save the day, allowing you to create fast and disposable build environments.

```sh
-rw-r--r-- 1 fs fs 8.6M 2024-12-22 17:33 linux-headers-6.12.6-test_6.12.6-1_amd64.deb
-rw-r--r-- 1 fs fs  19M 2024-12-22 17:33 linux-image-6.12.6-test_6.12.6-1_amd64.deb
-rw-r--r-- 1 fs fs 285M 2024-12-22 17:33 linux-image-6.12.6-test-dbg_6.12.6-1_amd64.deb
-rw-r--r-- 1 fs fs 1.4M 2024-12-22 17:33 linux-libc-dev_6.12.6-1_amd64.deb
```

Here are two examples with linux kernel image and zfs-linux backport.

<!--more-->

## First example: build a custom kernel

Let's start with the **Docker** image.
The base image is the same distribution and release as the target server.
Then install all the required software and create a non root user.
Ans that's it.

```docker
FROM debian:trixie-slim

RUN set -xe ;\
apt update -qqy ;\
  apt install -qqy fakeroot wget bzip2 build-essential build-essential bc \
    bison flex rsync libelf-dev libssl-dev dwarves git debhelper \
    libncurses-dev fakeroot wget bzip2 cpio kmod python3 ;\
  adduser --uid 1000 builduser

USER builduser
```

Now get the kernel and extract it.

```sh
$ wget -c https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.5.tar.xz
$ tar xfJ linux-6.12.5.tar.xz
```

On the target server, you can use the modlocalconfig to strip down the kernel configuration to the bare minimum and reduce build time.

```sh
$ wget -c https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.5.tar.xz
$ tar xfJ linux-6.12.5.tar.xz
$ cd linux-6.12.5
$ cp /boot/config-6.11.10-amd64 .config
$ make -j menucnfig
$ make localmodconfig
```

Now transfer the kernel config file from the target server.

```sh
$ rsync myserver:./tmp/kernel/linux-6.12.5/.config ./linux-6.12.5/
```

And build the Docker image and start the container with local folder as volume and run the build command

```sh
$ docker buildx build --pull --tag debbuilder:latest .
$ docker run -ti --rm -v .:/home/builduser/src debbuilder
builduser@92f887ec94c6:~$ cd ~/src/linux-6.12.5
builduser@92f887ec94c6:~$ make menuconfig # load the menu and exit to validate the config
builduser@92f887ec94c6:~$ make -j`nproc` bindeb-pkg LOCALVERSION=-test # build the Debian kernel packages
```

After a few minutes, you have your package files ready to be transferred and installed on your server.

```sh
$ rsync linux-headers-6.12.5-test_6.12.5-1_amd64.deb linux-image-6.12.5-test_6.12.5-1_amd64.deb linux-libc-dev_6.12.5-1_amd64.deb myserver:./tmp/kernel
$ ssh myserver
$ cd tmp/kernel
$ sudo dpkg -i linux-headers-6.12.5-test_6.12.5-1_amd64.deb linux-image-6.12.5-test_6.12.5-1_amd64.deb linux-libc-dev_6.12.5-1_amd64.deb
```

## Second example: backport zfs-linux package from Debian SID

Another example with backporting the [zfs-linux](https://packages.debian.org/source/sid/zfs-linux) package from [Debian](https://www.debian.org/) [Sid](https://www.debian.org/releases/sid/) to [Debian](https://www.debian.org/) [Trixie](https://www.debian.org/releases/trixie/).

A few things are added on the Docker image.
The sudo command allow to install build dependencies at run time, and the apt-src source to download package source files.

```docker
FROM debian:trixie-slim

ADD sid-src.sources /etc/apt/sources.list.d/sid-src.sources

RUN set -xe ;\
apt update -qqy ;\
  apt install -qqy fakeroot wget bzip2 build-essential build-essential bc \
    bison flex rsync libelf-dev libssl-dev dwarves git debhelper \
    libncurses-dev fakeroot wget bzip2 cpio kmod python3 vim sudo ;\
  adduser --uid 1000 builduser ;\
  echo 'builduser ALL=(ALL:ALL) NOPASSWD: ALL' > /etc/sudoers.d/builduser

USER builduser
```

The APT *deb-src* source in [deb822 format](https://repolib.readthedocs.io/en/latest/deb822-format.html).

*sid-src.sources*:
```text
Types: deb-src
URIs: http://ftp.ch.debian.org/debian
Suites: sid
Components: main contrib
```

Now create a new directory, start the container in it and install the [devscripts](https://packages.debian.org/search?keywords=devscripts) package

```sh
$ mkdir build-folder
$ cd $_
$ docker run -ti --rm -v .:/home/fs/src debbuilder
builduser@22c6ffb18ee4:/$ sudo apt install devscripts
```
Install the build dependencies, get the source and build the packages

```sh
builduser@22c6ffb18ee4:/$ cd ~/src
builduser@22c6ffb18ee4:~/src$ sudo apt build-dep zfs-linux
builduser@22c6ffb18ee4:~/src$ apt source zfs-linux
builduser@22c6ffb18ee4:~/src$ cd zfs-linux-2.2.7/
builduser@22c6ffb18ee4:~/src/zfs-linux-2.2.7$ debuild -b -uc -us
builduser@22c6ffb18ee4:~/src/zfs-linux-2.2.7$ 
```

And a few minutes later you have the package files ready to be installed.

```sh
$ ls -lh *.deb
-rw-r--r-- 1 fs fs  60K 2024-12-17 08:38 libnvpair3linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 108K 2024-12-17 08:38 libnvpair3linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  33K 2024-12-17 08:38 libpam-zfs_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  24K 2024-12-17 08:38 libpam-zfs-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  51K 2024-12-17 08:38 libuutil3linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  56K 2024-12-17 08:38 libuutil3linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 226K 2024-12-17 08:38 libzfs4linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 607K 2024-12-17 08:38 libzfs4linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  37K 2024-12-17 08:38 libzfsbootenv1linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  14K 2024-12-17 08:38 libzfsbootenv1linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 1.9M 2024-12-17 08:38 libzfslinux-dev_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 1.3M 2024-12-17 08:38 libzpool5linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 4.7M 2024-12-17 08:38 libzpool5linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  68K 2024-12-17 08:38 python3-pyzfs_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  61K 2024-12-17 08:38 pyzfs-doc_2.2.7-1_all.deb
-rw-r--r-- 1 fs fs 2.3M 2024-12-17 08:38 zfs-dkms_2.2.7-1_all.deb
-rw-r--r-- 1 fs fs  35K 2024-12-17 08:38 zfs-dracut_2.2.7-1_all.deb
-rw-r--r-- 1 fs fs  36K 2024-12-17 08:38 zfs-initramfs_2.2.7-1_all.deb
-rw-r--r-- 1 fs fs  28M 2024-12-17 08:38 zfs-test_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 654K 2024-12-17 08:38 zfs-test-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 552K 2024-12-17 08:38 zfsutils-linux_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 779K 2024-12-17 08:38 zfsutils-linux-dbgsym_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs  79K 2024-12-17 08:38 zfs-zed_2.2.7-1_amd64.deb
-rw-r--r-- 1 fs fs 110K 2024-12-17 08:38 zfs-zed-dbgsym_2.2.7-1_amd64.deb
```
