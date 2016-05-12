# Notes on making a Linux Distro

## Tools

First, you have to make sure LXC is installed.

```
sudo apt install lxc
```

LXC is a Linux container system that will allow us to create a specialized environment for creating our Linux distro (including building our own custom kernel, etc) without having to worry about messing up our existing kernel.

## Setup the Boot Creation Container

We'll create a container with Ubuntu Xenial 64 bit OS installed.  The `-n boot` names the container "boot", but you can choose to use whatever name you prefer.  Just remember to use that name instead of "boot" as you follow the rest of these instructions.

```
sudo lxc-create -t download -n boot -- -d ubuntu -r xenial -a amd64
```

This creates a container with the files located at `/var/lib/lxc/boot/` (but it's not accessible using a less privileged user, so you may need to `sudo su` first.)

At this point I make sure to have a couple of terminals open.  One is the "boot" terminal and the other is the "host" terminal.

Start the container and attach to it using the "boot" terminal.

```
boot:#
sudo lxc-start -n boot
sudo lxc-attach -n boot
```

Download a stable Linux kernel from [kernel.org](http://kernel.org) and copy it to the boot container.

Download a copy of [Syslinux](http://www.syslinux.org/wiki/index.php?title=Download) too.

(I downloaded to my Downloads folder using a web browser, but you can use wget if you prefer)

```
root@ubuntu:/#

cd/var/lib/lxc/boot/rootfs/root
cp /home/trichards/Downloads/linux-4.5.3.tar.xz .
tar -xvf linux-4.5.3.tar.xz
cp /home/trichards/Downloads/syslinux-4.04.tar.gz
tar -xvf syslinux-4.04.tar.gz
```

From your boot container, make the kernel.

```
root@boot
cd /root/linux-4.5.3
apt update
apt upgrade
apt install make gcc libncurses5-dev libelf-dev bc
make menuconfig
make bzImage
```

Create some directories to hold the ISO image master file tree.

```
mkdir CD_root
cd CD_root
mkdir isolinux
mkdir boot
mkdir install
cd ..
cp syslinux-4.04/core/isolinux.bin CD_root/isolinux/
```

Create a `isolinux.cfg` file.  You can see an example [here](http://www.syslinux.org/wiki/index.php?title=Isolinux.cfg) and instructions [here](http://www.syslinux.org/wiki/index.php?title=Config).

Here's what mine looks like.

```
display boot.txt
prompt 1
default aethos
# Default boot to Linux
label aethos
  menu label ^Start or install AethOS
  kernel /boot/bzImage
  append initrd=/boot/initrd.img
```


Create the initial file system that will reside on a ram disk.
[More background here.](http://www.thewireframecommunity.com/node/14)

Using the boot container, make a initfs directory.
```
root@boot:/root
mkdir initfs
cd initfs
mkdir {bin,sys,dev,proc,etc,lib,mount}
mkdir mount/cdrom
```

Create the device nodes. I used [this as an  example](https://lennartb.home.xs4all.nl/installdisk/node8.html).

```
mknod dev/console c 5 1
mknod dev/ram0 b 1 1
mknod dev/null c 1 3
mknod dev/tty1 c 4 1
mknod dev/tty2 c 4 2
mknod dev/sda b 8 0
mknod dev/sda1 b 8 1
mknod dev/sda2 b 8 2
mknod dev/sda3 b 8 3
mknod dev/sda4 b 8 4
mknod dev/sda5 b 8 5
mknod dev/sda6 b 8 6
mknod dev/sda7 b 8 7
mknod dev/sda8 b 8 8
mknod dev/sdb b 8 16
mknod dev/sdb1 b 8 17
mknod dev/sdb2 b 8 18
mknod dev/sdb3 b 8 19
mknod dev/sdb4 b 8 20
mknod dev/sdb5 b 8 21
mknod dev/sdb6 b 8 22
mknod dev/sdb7 b 8 23
mknod dev/sdb8 b 8 24
mknod dev/sr0 b 11 0
mknod dev/sr1 b 11 1
```

Copy BusyBox; I'm just using the one from my host machine because it's statically linked.
```
root@ubuntu:/var/lib/lxc/boot/rootfs/root/initfs/bin
cd /var/lib/lxc/boot/rootfs/root/initfs/bin
cp /bin/busybox .
ln -s busybox ash
ln -s busybox mount
ln -s busybox echo
ln -s busybox ls
ln -s busybox cat
ln -s busybox ps
ln -s busybox dmesg
ln -s busybox sysctl
ln -s busybox sh
ldd busybox
```

The last command (`ldd busybox`) should say "not a dynamic executable"; otherwise you have to copy the required libraries to `initfs/lib`.

For now we're only going to have a root login and everything will be executed as root, so it's safe to just link the bin and sbin directories.
```
root@boot:/root/initfs
ln -s bin sbin
```

Also, for now `rcS` is simple (put it in `/etc/init.d/rcS`).  Don't forget to make it executable.  [More info on rcS](http://unix.stackexchange.com/questions/56075/why-is-rcs-required-after-file-system-is-mounted-by-the-kernel)

```
#!/bin/sh
mount -n -t proc /proc /proc
mount -n -t sysfs none /sys
```

For some reason, `/etc/init.d/rcS` isn't being executed and the current setup is expecting an `/init` file.  My current setup is this:
```
#!/bin/ash
mount -t proc /proc /proc
mount -t sysfs none /sys
mount /dev/sr0 /mount/cdrom
echo
echo "initrd is running"
echo "Using BusyBox..."
echo
exec /bin/ash
```

Using the boot container, create the initrd image and copy that plus the boot image to the `CD_root/boot` directory.

```
root@boot:/root/initfs
find . | cpio -o -H newc | gzip > ../CD_root/boot/initrd.img
cp /root/linux-4.5.3/arch/x86_64/boot/bzImage /root/CD_root/boot/
```

Create the iso image (I did this from the host OS, but I think you can do this from the container).  After you create it, change the ownership and copy it to your home directory so you can burn it or whatever.

```
root@ubuntu:/var/lib/lxc/boot/rootfs/root
export AETHOS_VERSION=0.0.12
mkisofs -o aethos-$AETHOS_VERSION-amd64.iso -b isolinux/isolinux.bin \
  -c isolinux/boot.txt -no-emul-boot -boot-load-size 4 \
  -boot-info-table CD_root
chown trichards.trichards aethos-$AETHOS_VERSION-amd64.iso
cp aethos-$AETHOS_VERSION-amd64.iso /media/psf/Home/Downloads
```

## Notes after you've booted.

You can determine the location of blockdevices with partitions by using:
```
cat /proc/partitions
```

The cd-rom should be `/dev/sr0`, but if it's not then you should be able to find it in the `/proc/partitions` special file.  *Note that this only shows the major/minor device id's, so you'll have to do `mknod /dev/{dev} b {major} {minor}` to create the device before you can mount it.*
