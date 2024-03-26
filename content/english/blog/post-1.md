---
title: "How to generate a bootable ISO from a Docker image"
meta_title: ""
description: "Tutorial on how to turn a container into a bootable ISO"
date: 2024-03-07T05:00:00Z
image: "/images/image-placeholder.png"
categories: ["Application", "Data"]
author: "Agustin Chiappe Berrini"
tags: ["Linux", "docker"]
draft: false
---

Long ago, someone asked me to answer a very interesting question on Quora.
Thinking that it would make an interesting article by itself, I re-edited it into a Medium post.
The question poses a fun challenge. I will ignore boring follow-ups such as “why would you want to do that,” and just do
it.

My first intuition is to use the docker image as an initramfs and build an iso image by adding the other components.
We need the following:

1. To convert the image into an initramfs file.
2. To create an init program.
3. To get a kernel.
4. To get a bootloader.
5. To put it all together into an iso.

We will use tools that are common on any Linux, but we will need some weird programs:

- Docker, version 18.09.4-ce
- cpio version 2.12
- syslinux version 6.04.
- mtools version 4.0.23.
- git version 2.21.0

## Convert the image into an initramfs file and create the init program

In one terminal, we will execute:

```shell
$ docker run -it busybox /bin/sh
/ # touch IAMROOT
```

We will use that file as a mark to recognize the file system.
Now, without closing that session, in another terminal, execute:

```shell
$ docker ps | sed -n 2p | awk ‘{print $1}’ | xargs -I {} docker commit {} initramfs
$ docker image save initramfs initramfs.tar
```

The first command committed the last ran container into an image. The second saved the content of that image into a tar file.
Now:

```shell
$ tar xf initramfs.tar
$ ls
0a9af2d699eb71f4a3c1508ff39ed4cd7c00ed36a5cbfa383aacc931f455a395/
initramfs.tar
0dcec08843618e749e94ea72a835dd4481214407143285b4e5c57666f8b0f941.json
manifest.json
39c8a893588d75c746ab712c53edf61149745bca52dcc8d2e8e4385499ed166a/
repositories
$ cat repositories
{“initramfs”:{“latest”:”0a9af2d699eb71f4a3c1508ff39ed4cd7c00ed36a5cbfa383aacc931f455a395"}}
Those two folders are the layers of the image. We need to merge them. We should do it in order, so we should read the repositories file to find the last one.
$ mkdir root
$ tar xf 39c8a893588d75c746ab712c53edf61149745bca52dcc8d2e8e4385499ed166a/layer.tar -C root
$ tar xf 0a9af2d699eb71f4a3c1508ff39ed4cd7c00ed36a5cbfa383aacc931f455a395/layer.tar -C root/
$ mkdir root/proc root/sys
$ ls root/
bin dev etc home IAMROOT proc root tmp sys usr var
```

That looks correct! Note that we need to create the proc and sys folders because they were mounted as pseudo-filesystems in the container and were not part of the image.

## Create an init program

Now, we add a file in the root folder named init with the following content:

```shell
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
cat <<!
Boot took $(cut -d’ ‘ -f1 /proc/uptime) seconds
Welcome to your docker image based Linux.
!
exec /bin/sh
```

We just created our init program. It will be the process with PID 1. It mounts proc and sysfs (the two pseudo filesystems mentioned before), prints a message, and starts a shell. Take that systemd.
Finally, we will mark init as executable and package all the file system into a nice cpio file:

```shell
$ chmod +x root/init
$ cd root
$ find . -print0 | cpio — null -ov — format=newc | gzip -9 > ../initramfs.cpio.gz
$ cd ..
```

## Get a kernel

Now we need a kernel to run on our fancy file system. Let’s get to work:

```shell
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git # This will take a while. Get some coffee.
$ cd linux
$ make allnoconfig
$ make menuconfig
```

In the configuration interface, we mark the following:

```
64-bit kernel — -> yes
General setup — -> Initial RAM filesystem and RAM disk (initramfs/initrd) support — -> yes
General setup — -> Configure standard kernel features — -> Enable support for printk — -> yes
Executable file formats / Emulations — -> Kernel support for ELF binaries — -> yes
Executable file formats / Emulations — -> Kernel support for scripts starting with #! — -> yes
Device Drivers — -> Generic Driver Options — -> Maintain a devtmpfs filesystem to mount at /dev — -> yes
Device Drivers — -> Generic Driver Options — -> Automount devtmpfs at /dev, after the kernel mounted the rootfs — -> yes
Device Drivers — -> Character devices — -> Enable TTY — -> yes
Device Drivers — -> Character devices — -> Serial drivers — -> 8250/16550 and compatible serial support — -> yes
Device Drivers — -> Character devices — -> Serial drivers — -> Console on 8250/16550 and compatible serial port — -> yes
File systems — -> Pseudo filesystems — -> /proc file system support — -> yes
File systems — -> Pseudo filesystems — -> sysfs file system support — -> yes
```

And now, we compile it:

```shell
$ make -j8 > /dev/null
$ cp arch/x86_64/boot/bzImage ..
$ cd ..
```

Funny how it took longer to pull the code than to compile it, eh?
We created a simple version of the Linux kernel with only the bare minimum components to work.
Now we can check if what we did so far makes sense:

```shell
$ qemu-system-x86_64 -kernel bzImage -initrd initramfs.cpio.gz -nographic -append “console=ttyS0” -enable-kvm
Run /init as the init process
Boot took 0.18 seconds
Welcome to your docker image based linux.
/bin/sh: can’t access tty; job control turned off
/ # input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input2
clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x255dc4100df, max_idle_ns: 440795312434 ns
clocksource: Switched to clocksource tsc
/ # ls
IAMROOT dev home proc sys usr
bin etc init root tmp var
```

Aha, it works. It complains about not being able to use tty, but nothing we need to fix (right now).
Get a bootloader and put all together into an iso image
We are going to use syslinux as a bootloader because it is easy to install.

```shell
$ ls -lh bzImage initramfs.cpio.gz
-rw-r — r — 1 agustin users 1.2M Mar 29 20:13 bzImage
-rw-r — r — 1 agustin users 712K Mar 29 20:20 initramfs.cpio.gz
$ truncate -s100M bootable.iso
$ fdisk bootable.iso
Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only until you decide to write them.
Be careful before using the write command.
The device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x4d614621.
Command (m for help): o
Created a new DOS disklabel with disk identifier 0x14f4dcb2.
Command (m for help): n
Partition type
p primary (0 primaries, 0 extended, 4 free)
e extended (container for logical partitions)
Select (default p):
Using default response p.
Partition number (1–4, default 1):
First sector (2048–204799, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048–204799, default 204799):
Created a new partition 1 of type ‘Linux’ and of size 99 MiB.
Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 6
Changed type of partition ‘Linux’ to ‘FAT16’.
Command (m for help): a
Selected partition 1
The bootable flag on partition 1 is enabled now.
Command (m for help): p
Disk bootable.iso: 100 MiB, 104857600 bytes, 204800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x14f4dcb2
Device Boot Start End Sectors Size Id Type
bootable.iso1 * 2048 204799 202752 99M 6 FAT16
Command (m for help): w
The partition table has been altered.
Syncing disks.
```

We created an empty iso file, then we created one partition of type FAT16, and we marked it as bootable. 
There’s one problem, though: We have no easy way to access that partition, as bootable.iso1 doesn’t exist.
To solve it, We can use loop devices magic:

```shell
$ START=$((512 * 2048))
$ SIZE=”99M”
$ DEVICE=$(losetup -o $START — sizelimit $SIZE — show — find bootable.iso)
$ mkfs.fat -F 16 $DEVICE
```

Awesome. Now we have a partition already formatted with FAT16. Note that to create the loop device, we had to specify 
where the partition starts and ends. We do that using the information we got from fdisk: The size of a sector,the first sector of the partition, and the last one. Now we are ready to start installing syslinux.

```shell
$ syslinux -i $DEVICE
$ find / -name mbr.bin 2>/dev/null
/usr/lib/syslinux/bios/mbr.bin
$ dd bs=440 count=1 conv=notrunc if=/usr/lib/syslinux/bios/mbr.bin of=bootable.iso
$ mkdir disk
$ sudo mount $DEVICE disk
$ sudo cp bzImage initramfs.cpio.gz disk
```

Next, we have to configure syslinux. You should create a new file in the disk folder, called syslinux.cfg, and put the following content:

```
DEFAULT linux
LABEL linux
LINUX bzImage append console=ttyS0
INITRD initramfs.cpio.gz
```

Beautiful. We are telling syslinux what we told qemu before. And now, the final test:

```shell
$ sudo umount disk
$ losetup -d $DEVICE
$ qemu-system-x86_64 — enable-kvm — hda bootable.iso -nographic
Run /init as init process
Boot took 0.18 seconds
Welcome to your docker image based linux.
/bin/sh: can’t access tty; job control turned off
/ # input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input2
clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x255cd05bdcc, max_idle_ns: 440795290363 ns
clocksource: Switched to clocksource tsc
ls
IAMROOT dev home proc sys usr
bin etc init root tmp var
random: fast init done
/ #
```

And we did it!
If this was fun, try some of these follow-up exercises:

- The file system and kernel, together, add to 2MB. However, the image we created is 100MB (!!). Can you reduce the
  size to the minimum possible? What problems do you face? What if you try a different filesystem? What if you tried
  with an even simpler bootloader? How small can you make it?
- One use case for bootable devices is to recover broken systems. Create a docker container from the alpine Linux image;
  install standard recovery tools such as TestDisk, grub, parted, and photorec; and then follow the steps in this answer
  to have a useful bootable image.
- Can you add a program to install the filesystem in a disk and create a persistent version of our simple Linux?
- Our init program is small and lazy. Can you improve it? Start by fixing the warnings on boot, such as the tty complain.
  Then research about System 5 init and try to come up with something similar. Then, if you feel particularly evil, add
  a bunch of responsibilities that you wouldn’t expect from an init program: Logging, session, or network management.
