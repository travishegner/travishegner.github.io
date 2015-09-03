---
title: 'Ubuntu Server 9.04 Jaunty: open-vm modules fail to build'
author: travis.hegner
layout: post
permalink: /2009/05/ubuntu_server_904_jaunty_open-vm_modules_fail_to_build/
dsq_thread_id:
  - 2149846527
categories:
  - I.T.
  - Tutorials
---
I have decided to start upgrading from Ubuntu Server 8.10 Intrepid to Ubuntu Server 9.04 on all of my server machines. In 8.10, I was able to get the open-vm-tools to build and install, but they suffered from a bug which caused the vmware-guestd service to crash soon after starting. This was mildly acceptable and my servers have been running that way for a while, because I wanted the fix to be easily installed from the ubuntu repositories.

Well after waiting 6 months, with no fix in the repo&#8217;s, Canonical released Ubuntu Server 9.04. I upgraded a test server &#8220;sudo do-release-upgrade&#8221; and rebooted, only to find that the new vmware-guestd installs and runs, but the new open-vm modules fail to build properly with module-assistant (sigh). So I decided that I don&#8217;t want my <span class="caps">VM&#8217;</span>s to run without modules, because they are very important for the way <span class="caps">ESX</span>i handles memory allocation and management, but I still want to upgrade my 8.10 servers because having vmware-guestd not crash is a huge plus. vmware-guestd handles tasks like allowing the <span class="caps">ESX</span>i host to shutdown the linux guest safely, and synchronizing the VM clock with the host clock.

So after some searching, I came across [this open-vm patch][1] which allows the modules to build successfully. If you are not comfortable with patching it yourself, read on, and I&#8217;ll explain how I did it, and provide a pre-patched source archive for you to replace on your machine, in order to allow module-assistant to build the modules successfully.

Many people (myself included) are a bit weary when it comes to compiling and installing linux modules. Probably mostly due to lack of research, because if it&#8217;s like anything else in linux, it is very straght-forward and easy to understand. [ModuleAssistant][2] is an awsome tool for [debian][3] (and inherently debian-based) linux distros which nearly automates the process of compiling and installing modules. Assuming the code is correct and m-a is installed, we could build and install the open-vm modules and tools in one fell swoop by typing:

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo m-a a-i open-vm&lt;br />
</code>
  </p>
</blockquote>

But since we don&#8217;t get that lucky, we have to do a little work to get the modules installed and compiled properly. First, take all the necessary precautions and backup your data, or take a snapshot. Then, run the command from above to prepare the machine for the new modules. Go through the prompts accepting the changes, and eventually you will get an error stating that the build has failed. At this point, choose &#8220;stop&#8221; and you should be back at the command prompt. Navigate to &#8216;/usr/src&#8217; and list the contents.

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /usr/src && ls&lt;br />
</code>
  </p>
</blockquote>

You should see a source archive titled &#8216;open-vm.tar.bz2&#8217;, rename this archive to something else, and replace it with my [pre-patched open-vm.tar.bz2][4]. Also rename the &#8216;modules&#8217; directory.

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo mv open-vm.tar.bz2 open-vm.tar.bz2.orig&lt;br />
sudo mv modules modules.orig&lt;br />
sudo wget http://www.travishegner.com/stuff/open-vm.tar.bz2&lt;br />
</code>
  </p>
</blockquote>

Once the file has been replaced, re-run the auto-install command from above and the build should complete successfully. The modules and tools should also install. Reboot for good measure to make sure everything comes up like it&#8217;s supposed to.

You should now be running both the open-vm modules, and vmware-guestd without any issues. <span class="caps">AWSOME</span>! No more errant clocks, and even the &#8220;Shutdown Guest&#8221; and &#8220;Restart Guest&#8221; scripts work like they are supposed to.

 [1]: http://gist.github.com/raw/101730/a525d1eb140661bf4067ae21adbb408856042780/vmhgfs-2008-11-18-jaunty-lenny.diff
 [2]: http://wiki.debian.org/ModuleAssistant
 [3]: http://www.debian.org/
 [4]: http://www.travishegner.com/stuff/open-vm.tar.bz2