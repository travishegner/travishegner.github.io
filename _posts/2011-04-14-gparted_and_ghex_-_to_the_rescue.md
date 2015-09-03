---
title: 'gparted, dd, and ghex2 &#8211; <del>Oh My!</del> To The Rescue!'
author: travis.hegner
layout: post
redirect_from: /2011/04/gparted_and_ghex_-_to_the_rescue/
dsq_thread_id:
  - 2149845132
categories:
  - I.T.
  - Tutorials
  - Ubuntu
---
So, I was messing around with an XP virtual machine that was out of disk space. For some dumb reason, I thought 4GB of hard disk space was plenty for XP when I built it. Well here we are a few years later, and M$ has release so many service packs and updates that the machine flat out won&#8217;t work with that small of a disk. 

<div>
</div>

<div>
  No biggie, it&#8217;s a virtual machine, just shut it down, and add more space to the disk. So it boots back up and there is the extra space, right at the end of my primary OS disk. Since I&#8217;m losing my touch with Microsoft products, I forgot the simple fact that XP <em>will not expand a partition to fill the disk, even in it&#8217;s native file system!</em>&nbsp;What a joke right? So I think, &#8220;I&#8217;ve heard those dynamic disks are a bit more flexible, I&#8217;ll try that&#8221;. I convert the disk to dynamic, which would have worked had this not been an OS disk. OK, so you can extend a non-OS disk but not an OS one? Great.
</div>

<div>
</div>

<div>
  Next, I boot with an Ubuntu live CD. Now we should be able to get some work done, even though NTFS is proprietary. So I fire up my trusty gparted, tell it to extend the disk and BAM! FAILED! Gparted doesn&#8217;t support dynamic disks. Doesn&#8217;t that figure, if only I had just booted with linux in the first place. How was I supposed to know that M$ can&#8217;t correctly deal with it&#8217;s own NATIVE FILE SYSTEM! BAH!
</div>

<div>
</div>

<div>
  So after about 10 tutorials that begin &#8220;To convert a dynamic disk to a basic disk, you must first delete any partitions that exist&#8230;&#8221; I found this one,&nbsp;<a href="http://www.wilderssecurity.com/showthread.php?t=191006">http://www.wilderssecurity.com/showthread.php?t=191006</a>. So you mean to tell me that converting it to dynamic simply flips a single byte on the disk? I&#8217;m sure it&#8217;s more complicated than that, but if I haven&#8217;t written significant amounts of data to the disk, it should be safe. This tutorial utilizes a windows app, and is not an OS disk. So I had to do a couple of things differently to make it work for me.
</div>

<div>
</div>

<div>
  First, from my live cd, I tried opening the disk directly with ncurses-hexedit. Looks like a great program, but it&#8217;s kind of hard to use, and I even consider myself advanced in linux CLI. I couldn&#8217;t figure out how to actually modify the contents of the disk. So, in steps dd:
</div>

<div>
  </p> 
  
  <blockquote class="code">
    <p>
      <code>&lt;br />
dd if=/dev/sda of=temp.out bs=1 count=500&lt;br />
</code>
    </p>
  </blockquote>
</div>

Since the byte I had to flip is so close to the beginning of the drive, I just grabbed the first 500 bytes into a temporary file. Open the file in ghex2, change the byte at 0x000001C2 from 42 to 07, then save the file. Then we dd it right back on top of the disk. *I shouldn&#8217;t have to mention here that you&#8217;d better have a backup. This is definitely a half hacked, jacked up way of doing things. Luckily for me this machine isn&#8217;t of supreme importance. It only controls the physical access to our entire corporate building.*

<meta http-equiv="content-type" content="text/html; charset=utf-8" />

<meta http-equiv="content-type" content="text/html; charset=utf-8" />

<div>
</div>

<blockquote class="code">
  <p>
    <code>&lt;br />
dd if=temp.out of=/dev/sda&lt;br />
</code>
  </p>
</blockquote>

<div>
  Boot windows back up and low and behold, our OS disk is back to basic. Then we boot back into our live cd, fire up our not as trusty anymore gparted, and BAM, IT WORKS! Great! Now I can install the 150 back-logged updates, and upgrade the vmware tools! Just another day at the office.
</div>