+++
date = '2024-11-28T11:00:00+01:00'
draft = false
title = 'Shrink Debian Docker Image'
keywords = ['open source', 'docker', 'debian', 'vagrant']
+++

## 1.8Gb docker image

As I encountered some ruby problem with [Vagrant](https://www.vagrantup.com/) on my [Archlinux](https://archlinux.org/)
laptop, I decided to use Docker as a workaround. 

```sh {class="code-overflow"}
$ vagrant
/usr/lib/ruby/3.3.0/rubygems/specification.rb:2245:in `raise_if_conflicts': Unable to activate vagrant_cloud-3.1.1, because rexml-3.3.2 conflicts with rexml (~> 3.2.5) (Gem::ConflictError)
        from /usr/lib/ruby/3.3.0/rubygems/specification.rb:1383:in `activate'
        from /usr/lib/ruby/3.3.0/rubygems/core_ext/kernel_gem.rb:62:in `block in gem'
        from /usr/lib/ruby/3.3.0/rubygems/core_ext/kernel_gem.rb:62:in `synchronize'
        from /usr/lib/ruby/3.3.0/rubygems/core_ext/kernel_gem.rb:62:in `gem'
        from /opt/vagrant/embedded/gems/gems/vagrant-2.4.2/bin/vagrant:17:in `block in <main>'
        from /opt/vagrant/embedded/gems/gems/vagrant-2.4.2/bin/vagrant:16:in `each'
        from /opt/vagrant/embedded/gems/gems/vagrant-2.4.2/bin/vagrant:16:in `<main>'
```
So let's create a Docker image, using a debian slim image and just install vagrant and some vagrant plugins.

After a two minutes build you have your image that is ... 1.8Gb large !

```sh {class="code-overflow"}
$ docker image ls vagrant:test
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
vagrant      test      05c7aa145055   27 seconds ago   1.8GB
```

What happenned exactly ?

```sh {class="code-overflow"}
$ docker run -ti --rm debian:bookworm-slim
# apt update
# apt install vagrant
<...>
0 upgraded, 679 newly installed, 0 to remove and 1 not upgraded.
Need to get 454 MB of archives.
After this operation, 1852 MB of additional disk space will be used.
Do you want to continue? [Y/n] n
Abort
```

679 new packages when you want to install just one, that is insane. 

Let's dig in with the **Debug::pkgDepCache::AutoInstall** apt debug option.

```text {class="code-overflow"}
  Installing vagrant-libvirt:amd64 as Recommends of vagrant:amd64
    Installing libguestfs-tools:amd64 as Recommends of vagrant-libvirt:amd64
      Installing libguestfs0:amd64 as Depends of libguestfs-tools:amd64
        Installing supermin:amd64 as Depends of libguestfs0:amd64
          Installing linux-image-amd64:amd64 as Recommends of supermin:amd64
            Installing linux-image-6.1.0-28-amd64:amd64 as Depends of linux-image-amd64:amd64
              Installing kmod:amd64 as Depends of linux-image-6.1.0-28-amd64:amd64
                Installing libkmod2:amd64 as Depends of kmod:amd64

        Installing qemu-system-x86:amd64 as Depends of libguestfs0:amd64
          Installing qemu-system-common:amd64 as Depends of qemu-system-x86:amd64
            Installing libspice-server1:amd64 as Depends of qemu-system-common:amd64
              Installing gstreamer1.0-plugins-good:amd64 as Recommends of libspice-server1:amd64
                Installing libsoup2.4-1:amd64 as Depends of gstreamer1.0-plugins-good:amd64
                  Installing glib-networking:amd64 as Depends of libsoup2.4-1:amd64
                    Installing gsettings-desktop-schemas:amd64 as Depends of glib-networking:amd64
                      Installing dconf-gsettings-backend:amd64 as Depends of gsettings-desktop-schemas:amd64
                        Installing dconf-service:amd64 as Depends of dconf-gsettings-backend:amd64
                          Installing dbus-user-session:amd64 as Depends of dconf-service:amd64
                            Installing libpam-systemd:amd64 as Depends of dbus-user-session:amd64
                              Installing systemd:amd64 as Depends of libpam-systemd:amd64
                                Installing libfdisk1:amd64 as Depends of systemd:amd64
```

So **libguestfs0** depends on **supermin** that recommands **linux-image-amd64** and now you have installed a kernel image, useless for a Docker image.

Same thing with **libspice-server1** that recommends **gstreamer1.0-plugins-good** lead by a dependencies cascase to install **systemd**

That is a total a 85 *Recommends of* that lead to 679 packages to be installed.

## Solution

The problem is that recommended packages are installed and cause a dependency cascade. The man page of [apt-get](https://manpages.debian.org/bookworm/apt/apt-get.8.en.html) give the solution the **-no-install-recommends** option.

Let's see what is the impact.

```sh {class="code-overflow"}
$ docker run -ti --rm debian:bookworm-slim
# apt update
# apt install --no-install-recommends vagrant
<...>
0 upgraded, 86 newly installed, 0 to remove and 1 not upgraded.
Need to get 34.5 MB of archives.
After this operation, 145 MB of additional disk space will be used.
Do you want to continue? [Y/n] n
Abort
```

Nice ! From 1.8Gb to 145Mb, that more than 12 times less !

Now I can update the Dockerfile

```docker {class="code-overflow"}
FROM debian:bookworm-slim

RUN set -ex ;\
  cd tmp ;\
  apt -yqq update ;\
  apt -yqq install --no-install-recommends bash vagrant vagrant-libvirt vagrant-sshfs ;\
  apt -yqq clean ;\
  adduser --uid 1000 --disabled-password vagrant ;\
  mkdir /data

WORKDIR /data
USER vagrant

ENTRYPOINT [ "/usr/bin/vagrant" ]
```

And the result is much more smaller.

```sh {class="code-overflow"}
$ docker image ls vagrant:test
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
vagrant      test      6265e705f81d   4 hours ago   245MB
```

The Dockerfile in available on [github.com/arcenik/vagrant-docker](https://github.com/arcenik/vagrant-docker)
and the Docker image on [Docker Hub](https://hub.docker.com/repository/docker/francois75/vagrant/general)
