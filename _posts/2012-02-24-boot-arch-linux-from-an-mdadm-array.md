---
title: Boot Arch Linux from an mdadm array
author: travis.hegner
layout: post
permalink: /2012/02/boot-arch-linux-from-an-mdadm-array/
dsq_thread_id:
  - 2149845049
categories:
  - Arch Linux
  - I.T.
  - Tutorials
---
This method will allow you to configure your `/boot` directory or partition on a raid 0, 5, or 6. This was not previously possible before grub2. The code in this write-up was tested against a virtual machine with 2 20GB hard disks, with a couple of raid 0 partitions.

First, Boot your arch install image. Once running in your live environment, configure your partitions. It&#8217;s *very* important to remember that your first partition must begin further into the disk than the typical default of 2048 sectors, 4096 worked for me. My test box has the following partition table on each disk.

<pre>[root@archiso ~]# fdisk -l /dev/sda

Disk /dev/sda: 21.5 GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders, total 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x2a8e3edb

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        4096    37752831    18874368   fd  Linux raid autodetect
/dev/sda2        37752832    41943039     2095104   fd  Linux raid autodetect</pre>

Next create your arrays.

<pre>mdadm --create /dev/md0 --raid-devices=2 --level=0 /dev/sda1 /dev/sdb1
mdadm --create /dev/md1 --raid-devices=2 --level=0 /dev/sda2 /dev/sdb2</pre>

Once your arrays are set, run through your normal arch setup process, skipping all of the bootloader steps. Remember on the &#8220;Prepare Hard Drives&#8221; step to choose &#8220;Manually configure block devices, filesystems, and mountpoints&#8221;, and choose the appropriate /dev/mdX devices.

After the install is complete, we have a few more steps to get grub2 installed. Make sure you&#8217;ve got networking.

<pre>dhcpcd eth0</pre>

Prepare your installed root, and chroot into it.

<pre>cp /etc/resolv.conf /mnt/etc/resolv.conf 
mount -o bind /proc/ /mnt/proc/
mount -o bind /sys/ /mnt/sys/
mount -o bind /dev/ /mnt/dev/
chroot /mnt/</pre>

Now that we are essentially running within our installed environment, we can install grub2.

<pre>pacman -Sy grub2</pre>

Ignore the pacman upgrade request, and allow pacman to remove the &#8220;grub&#8221; package, as we don&#8217;t need it. Next, prepare your mdadm.conf file.

<pre>mdadm --examine --scan &gt; /etc/mdadm.conf</pre>

Add &#8220;mdadm&#8221; to the &#8220;HOOKS&#8221; array in your `/etc/mkinitcpio.conf`. As of this writing, a bug prevents your `/boot/grub/grub.cfg` from generating correctly unless you uncomment the line &#8220;GRUB\_TERMINAL\_OUTPUT=console&#8221; in your `/etc/default/grub`. Generate your `/boot/grub/grub.cfg` and install grub2 to every device in your array.

<pre>grub-mkconfig -o /boot/grub/grub.cfg
grub-install /dev/sda
grub-install /dev/sdb</pre>

Re-generate your init image.

<pre>mkinitcpio -p linux</pre>

You should see a message go by saying

<pre>Custom /etc/mdadm.conf file will be used in initramfs for assembling arrays.</pre>

If you don&#8217;t see this message, make sure you didn&#8217;t miss any steps above. You should be able to boot from any disk in your array into your new arch install.

If you hit busybox, boot back into your live environment, chroot back into your installed environment, modify your /etc/default/grub to uncomment &#8220;GRUB\_DISABLE\_LINUX_UUID=true&#8221;, and re-run the grub-mkconfig command from above. I&#8217;ve seen a udev/UUID bug that sometimes prevents your initramfs mdadm from correctly assembling your arrays.

Most of the information in the article was pieced together from [https://wiki.archlinux.org/index.php/Software\_RAID\_and_LVM][1] and <https://wiki.archlinux.org/index.php/GRUB2> and maybe a couple of other places.

 [1]: https://wiki.archlinux.org/index.php/Software_RAID_and_LVM