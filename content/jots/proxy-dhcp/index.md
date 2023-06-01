---
date: "2019-05-26"
draft: false
showpagemeta: true
title: "Quick setup of a proxyDHCP with dnsmasq and PXE boot"
imgurl: "https://unsplash.com/photos/xyDQNmT6vSs"
---

> Jotting down some notes on how to setup a proxyDHCP using dnsmasq in order to boot a device from a network rather than from a local drive.

{{< figure src="/jots/proxy-dhcp/kvistholt-photography-oZPwn40zCK4-unsplash.jpg" caption="Photo by [Kvistholt Photography](https://unsplash.com/photos/oZPwn40zCK4)" link="https://unsplash.com/photos/oZPwn40zCK4" >}}

How to netboot a device in a few simple steps.

Since we will be relying on PXE, we need to ensure that our booting device has a network card that runs a PXE ROM, i.e it runs a firmware that implements some necessary internet protocols: UDP/IP, DHCP and TFTP.

We then need a DHCP server that is capable of replying to our device with both:

  1. IP address, IP mask, etc
  2. a bootstrap program (the PXE part of things)

Not every router or DHCP server allows configuring (2.), in which case we may setup a proxyDHCP, for instance with `dnsmasq`.

### dnsmasq

The `dnsmasq` service should be running on a machine sharing the network with the booting devices. It may be running in the host itself, in a virtual machine or even from a docker container.

In the following examples the `dnsmasq` service will be running from inside a container and configuration passed by a volume mount. So, in short we need the following Dockerfile:

```Docker
FROM alpine:latest
RUN apk --no-cache add dnsmasq
WORKDIR /tftpboot
CMD ["dnsmasq", "-d"]
```

which can be built with:

```bash
docker build -f Dockerfile -t proxydhcp .
```

and run with:

```bash
docker run \
    --rm -it \
    --net=host \
    --cap-add=NET_ADMIN \
    -v ${PWD}/dnsmasq.conf:/etc/dnsmasq.conf \
    -v ${PWD}/tftpboot:/tftpboot \
    proxydhcp
```

> **ℹ️ Note:**
> don't run just yet, we need to configure `tftpboot/` directory.

dnsmasq should be configured (`dnsmasq.conf`) with:

```ini
port=0
log-dhcp
dhcp-range=192.168.9.0,proxy
dhcp-boot=pxelinux.0
pxe-service=x86PC,'Network Boot',pxelinux
enable-tftp
tftp-root=/tftpboot
```

> **ℹ️ Note:**
> `dhcp-range` should be set with the correct network address.

Create a directory named `tftpboot`:

```bash
mkdir -p tftpboot/
```

Two examples are covered, booting from:

- Ubuntu 18.04 netboot
- XenServer from iso


### Ubuntu 18.04 netboot example config

Download Ubuntu 18.04 netboot from here [netboot_url], and decompress it into `tftpboot/`:

    cd tftpboot/
    wget http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/netboot.tar.gz
    tar xf netboot.tar.gz && rm netboot.tar.gz
    cd -

`tftpboot/` should look like this:

    tftpboot/
    ├── ubuntu-installer/
    ├── ldlinux.c32 -> ubuntu-installer/amd64/boot-screens/ldlinux.c32
    ├── pxelinux.0 -> ubuntu-installer/amd64/pxelinux.0
    ├── pxelinux.cfg -> ubuntu-installer/amd64/pxelinux.cfg
    └── version.info

And we should now be able to run our docker container as mentioned above and boot the PXE ROM machine.

[netboot_url]: http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/netboot.tar.gz


### XenServer ISO example config

Download XenServer Free from [xenserver.org]

[xenserver.org]: https://xenserver.org

and then mount bind the iso file:

    sudo mkdir -p /mnt/xeniso/
    sudo mount -o loop XenServer-7.6.0-install-cd.iso /mnt/xeniso/

Copy `mboot.c32`, `menu.c32` and `pxelinux.0` from inside `/mnt/xeniso/boot/pxelinux/` to `tftpboot/`:

    cp /mnt/xeniso/boot/pxelinux/* tftpboot/

Create the directories `tftpboot/pxelinux.cfg/` and `tftpboot/xenserver/`

    mkdir -p tftpboot/pxelinux.cfg/
    mkdir -p tftpboot/xenserver/

and copy all the contents from `/mnt/xeniso/boot` to `tftpboot/xenserver/`:

    cp /mnt/xeniso/boot/* tftpboot/xenserver/
    cp /mnt/xeniso/install.img tftpboot/xenserver/

and create the file `tftpboot/pxelinux.cfg/default` with content:

```
default xenserver
label xenserver
kernel mboot.c32
    append /tftpboot/xenserver/xen.gz dom0_max_vcpus=2 dom0_mem=1024M,max:1024M com1=115200,8n1 console=com1,vga ---  /tftpboot/xenserver/vmlinuz xencons=hvc console=hvc0 console=tty0 ---  /tftpboot/xenserver/install.img
```

Finally expose `/mnt/xeniso` through http for instance by running:

    cd /mnt/xeniso
    sudo python3 -m http.server 80

Run the container and boot the PXE device. Follow the installation and instructions and configure installation from http, pointing to the python http server setup above.
