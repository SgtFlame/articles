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

Download a stable Linux kernel from kernel.org and copy it to the boot container.

Download a copy of Syslinux too.

http://www.syslinux.org/wiki/index.php?title=Download


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

Create a `isolinux.cfg` file.  You can see examples here:
http://www.syslinux.org/wiki/index.php?title=Isolinux.cfg
And instructions here:
http://www.syslinux.org/wiki/index.php?title=Config

```
display boot.txt
prompt 1
default aethos
# Default boot to Linux
label aethos
  menu label ^Start or install AethOS
  kernel /boot/vmlinuz
  append initrd=/boot/initrd.img
```


Create the initial file system that will reside on a ram disk.
(instructions here: http://www.thewireframecommunity.com/node/14)

Using the boot container, make a initfs directory.
```
root@boot:/root
mkdir initfs
cd initfs
mkdir {bin,sys,dev,proc,etc,lib}
```

Create the device nodes:
```
mknod dev/console c 5 1
mknod dev/ram0 b 1 1
mknod dev/null c 1 3
mknod dev/tty1 c 4 1
mknod dev/tty2 c 4 2
```

Copy BusyBox; I'm just using the one from my host machine because it's statically linked.
```
root@ubuntu:/var/lib/lxc/boot/rootfs/root/initfs/bin
cd /var/lib/lxc/boot/rootfs/root/initfs/bin
cp /bin/busybox .
ldd busybox
```

The last command (`ldd busybox`) should say "not a dynamic executable"; otherwise you have to copy the required libraries to `initfs/lib`.

For now we're only going to have a root login and everything will be executed as root, so it's safe to just link the bin and sbin directories.
```
root@boot:/root/initfs
ln -s bin sbin
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
mkisofs -o aethos-0.1-amd64.iso -b isolinux/isolinux.bin \
  -c isolinux/boot.txt -no-emul-boot -boot-load-size 4 \
  -boot-info-table CD_root

chown trichards.trichards aethos-0.1-amd64.iso
cp aethos-0.1-amd64.iso /home/trichards/

```
