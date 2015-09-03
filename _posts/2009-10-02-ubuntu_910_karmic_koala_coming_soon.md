---
title: Ubuntu 9.10 Karmic Koala Coming Soon!
author: travis.hegner
layout: post
redirect_from: /2009/10/ubuntu_910_karmic_koala_coming_soon/
dsq_thread_id:
  - 2149845536
categories:
  - I.T.
  - Ubuntu
---
<center>
  <br /> <br />
</center>

  
  
I am pleased to report that the 9.10 release of Ubuntu is available for Beta testing. I have had the Alpha running on a test machine for several weeks now with daily updates applied. My First impressions of the release are pleasing, but I haven&#8217;t done too much with it yet. The first thing you&#8217;ll notice when starting your new 9.10 machine is that the new kernel now seems to have some major video improvements. The screen resolution is automatically detected on my test machine, and the native resolution is applied even before the splash screen. If you use the virtual terminals (ctrl-alt-f1 through f6), then this is a awsome benefit, as the virtual terminals now run in your monitors native resolution, and switching between them is lightning fast. 

The next thing you&#8217;ll notice is the new splash screen in 9.10. This new piece of work is nothing short of awsome looking. The default user selection/logon screen is a bit boring though. First, I don&#8217;t like the &#8220;user picker&#8221; style login screens. For an attempting intruder, half the battle of gaining access is done for you, you don&#8217;t have to guess the username. It&#8217;s also not quite as fancey as the new splash screen we just drooled over. I very much prefered the 9.04 default login screen, perhaps they&#8217;ll change it before the final release. A quick glance at System->Administration->Login Screen tells me that they&#8217;ve taken away the ability to change the default login screen. This is dissapointing for the reasons I stated above, and for those that like to completely customize their system.

The third observation you&#8217;ll have is the default color scheme. The ubuntu team has finally let go of the orange title bars in gnome which have served so well for the last&#8230; well&#8230; since I started using ubuntu anyway. The have moved to a much darker brown, which is fine with me, I prefer most dark color schemes anyway. This is a very customizeable option anyway, and if you don&#8217;t like it, it&#8217;s very easy to change under System->Preferences->Appearance. For those that really want to spruce up ubuntu, or any gnome based distribution, have a look at http://www.gnome-look.org/.

For those of us that use ubuntu in a corporate environment which runs Microsoft Exchange 2007, I have some good news, and some bad news. The long awaited, and long sought after, evolution-mapi plugin is installable, and you can create and authenticate to your exchange mailbox with the native mapi library. And the bad news? It (still) doesn&#8217;t seem production ready! I have my account configured and was able to use it for a few minutes the first time, but when loading certain messages, evolution crashes (completely disappears from the screen) and you lose anything not saved. Worse yet, when I open evolution now, it crashes (disappears) within 5 to 10 seconds, rendering the app useless until I delete my ~/.evolution folder to delete my configuration. It&#8217;s so close, yet so far away!

One thing I don&#8217;t understand is why someone hasn&#8217;t tackled this evolution to exchange problem using EAS or Exchange Active-Sync. Exchange has a web service that runs, which is designed for mobile applications, and any windows based mobile phone can link seemlessly and download any and all emails, tasks, and calander apppointments. Even Google&#8217;s android, and the new Palm Pre can utilize this service to accomplish the same thing opensource users have been asking for years. Exchange mailbox access, from a non microsoft client. I realize that the actual API is probably licensed, and costs money to use, but I know there are some brilliant people out their who can design their own API, open source of course, that sends requests and responses to the EAS service the same way the microsoft licensed API would. It&#8217;s not much different than mimicing MAPI the way the new mapi libraries do, you&#8217;ll just be mimicing EAS instead. Oh well, maybe one day it will work.

That is about as far as I&#8217;ve taken my 9.10 box for a test drive. Only around the block. When the 9.10 final is released, I will be installing it on my production workstation, and reporting on all the goodies that I&#8217;ve been waiting to be fixed&#8230; Like nvidia support for xrandr, so I can get rid of the deprecated xinerama.

Until Next Time!