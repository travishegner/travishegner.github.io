---
title: 'You did WHAT with iptables? (Part 3: The Actual Solution)'
author: travis.hegner
layout: post
redirect_from: /2014/01/you-did-what-with-iptables-part-3-the-actual-solution/
dsq_thread_id:
  - 2149814923
categories:
  - I.T.
---
If you didn&#8217;t guess by the title, this is Part 3 in a whole series about abusing iptables. It&#8217;d be wise to read [Part 1][1], then [Part 2][2], then come back here.

If you are looking for a copy/paste answer, you won&#8217;t find it. The actual solution is spattered throughout all three parts to tell a story of triumph over data packets. A story of solving a Layer 7 problem with Layer 4 technologies. Some of the code you&#8217;ll find is not part of the end result, but shows the thought process I used to make it to a final solution. This is intentional to make you think about what you are actually doing before you copy and paste my highly specialized, for my environment only, commands and code into your environment and cause you to completely ruin your whole network, and eventually lose your job. Your Welcome.

> Nevertheless, the information contained in all three parts of this article, as well as in every other article on this blog, is purely for entertainment and informational purposes only. If you use this information and break something, then I&#8217;m not responsible. Just go fix it and leave me alone.

Good, now that we have that settled. Let&#8217;s get on with it. Now, where did we leave off? Oh&#8230; right&#8230; starting over.

I contemplated putting a media proxy between my UCM and the phone provider, but there was no guarantee that the media proxy would actually re-originate the RTP stream since each leg of the call would be using the same codec. There was also no guarantee that the (open source, of course) media proxy would play nice with the whole broken source port thing anyway. Plus that would have been a lot of work since I had no way to know before injecting a media proxy that the call was being held or resumed. And I&#8217;ve worked really, really hard to prevent my users voice packets to come back through the VPN, into my data center&#8217;s internet connection, and right back out again to the provider. I&#8217;m not going to start now. Besides, that would destroy the voice quality that my users have come to expect.

I contemplated standing up another instance of UCM in my cluster. A colossal waste of compute resources in my virtual environment just to have a dedicated MOH source. That way I wouldn&#8217;t have to do any detection, and I could blindly SNAT the source port and go on with life. But then I remembered that this would have the same problem with connection tracking as the current iptables mess that I had created.

So my best options were my own custom kernel module to add a stateless SNAT sort of target, or to look into this [QUEUE][3] target that I had seen some references to. At this point all I knew was that the QUEUE target made the data packets available in <a href="http://en.wikipedia.org/wiki/User_space" target="_blank">user-land</a>. Alright, this looks promising. I quickly learned that [NFQUEUE][4] is basically a more robust version of QUEUE, so I change gears and headed in that direction. With everything learned so far, I simply needed to figure out how to get the packet out of the queue, modify the source address and port, then send it back to the kernel. Sounds easy right? Sort of&#8230; The [standard way][5] to do this is to write a pure C program against libnetfilter_queue. Great, that is almost as hard as a kernel module. Possible, but not something that *I* can do overnight. Then I keep reading down that link and I find [nfqueue-bindings][6]. Awesome! Someone has written perl libraries which will allow me to connect to the queue, grab packets, issue verdicts, and even send a modified packet back through. PERFECT!

So I change my MOH-BLOCK chain to send packets destined for a host in the MOH set to `-j NFQUEUE --queue-num 0` instead of `-j ACCEPT`.

Then I copy the <a href="https://github.com/TrustRouter/nfqueue-bindings/blob/master/examples/rewrite.pl" target="_blank">rewrite.pl</a> example from the nfqueue-bindings project on github.com, and hack it up to do what I need. I fire up the script as root, and after a couple of snags (make sure you load the nfnetlink_queue kernel module), it was off and running.

The outbound MOH packets from my UCM are marked with NOTRACK, then queued for my perl script to statelessly flip the source ip and port. Music on hold is heard in every call scenario I can think to test, no matter how many times the caller is placed on hold. It is a sloppy and beautiful thing.

I know what you are thinking (not really), so I&#8217;ll try and answer those questions now.

**&#8220;What happens to the packets coming from the caller while they are on hold?&#8221;**  
Since the outbound packets which match that &#8220;connection&#8221; aren&#8217;t tracked as a connection at all, they hit the default rule of the forward table and are dropped. I value our customers privacy, so I won&#8217;t do it, but wouldn&#8217;t it be neat to capture those packets and store them somewhere. Then we could go in and decode those RTP streams to hear what people are actually saying while they are on hold. Sounds like the beginning of what could be a successful, and hilarious, website.

**&#8220;How well will this solution scale?&#8221;**  
I don&#8217;t know. How much CPU does your firewall have available? In my tests, I&#8217;m seeing about 1% CPU per MOH stream on a 2.4GHz processor. I have only ever seen 7 simultaneous MOH streams in my environment, so I don&#8217;t have anything to worry about currently.

**&#8220;That&#8217;s nothing. I need it to scale higher than that.&#8221;**  
Great. I&#8217;m happy for you. Write a pure C daemon against libnetfilter_queue, and that should do better.

**&#8220;Pffft. Child&#8217;s play&#8230; I need more!&#8221;**  
OK, then. Write a custom kernel module to do stateless nat/pat/pat-sat-on-(hat|cat|bat). This will prevent the data from having to be copied from kernel space to user-land and back. This should prove to be a significant performance increase. Please let me know if you do, and if it&#8217;s open source, because I could really simplify my configuration with something like that.

**&#8220;Wait a minute. Didn&#8217;t you say that your inbound local calls worked fine? If this is the solution, then how did those work in the first place?&#8221;  
**Oh, you were paying attention. Good Job, I like that. That is a funny story really. In all of my testing and analyzing and ruling out certain possibilities based on the fact *that* particular scenario worked, I discovered a large security hole in my providers network. Ironically, what worked only did so because of a broken provider configuration. Wrap your head around that. Basically, it seems, that calls initiated from their network outbound, don&#8217;t seem to have any firewall protection on the media gateways terminating the RTP stream. What that means is that given a known ip/port which is currently receiving an RTP stream for a currently on-going voice call, I can forcibly inject my own RTP stream **regardless of what the source ip address and port are**. Depending on the packet timing, you can hear the injected audio clearly, otherwise it comes across as a jumbling of the two streams mixed, or as just garbage. In any case, it means that a random internet stranger has the power to interrupt their customer&#8217;s (or their customer&#8217;s customer&#8217;s rather) phone calls. I notified my provider in one of my trouble tickets with a detailed account of what I believed was happening, and I was told that &#8220;they&#8217;ll look into it&#8221;. As of this writing (one week later), they have yet to fix it. *Figures*.

Well there you have it. One man&#8217;s account of a seemingly endless battle with disparate SIP implementations, which ultimately ended up being <del>solved</del> worked around with perl. Isn&#8217;t everything done in perl a workaround anyway?

No, I won&#8217;t <a href="http://xkcd.com/664/" target="_blank">sync your outlook with your new phone</a>.

 [1]: http://travishegner.com/2014/01/you-did-what-with-iptables-part-1-the-problem/
 [2]: http://travishegner.com/2014/01/you-did-what-with-iptables-part-2-the-almost-solution/
 [3]: http://www.linuxtopia.org/Linux_Firewall_iptables/x4501.html
 [4]: http://www.iptables.info/en/iptables-targets-and-jumps.html#NFQUEUETARGET
 [5]: https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/
 [6]: https://www.wzdftpd.net/redmine/projects/nfqueue-bindings/wiki