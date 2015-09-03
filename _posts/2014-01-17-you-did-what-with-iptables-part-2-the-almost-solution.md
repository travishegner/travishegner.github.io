---
title: 'You did WHAT with iptables? (Part 2: The Almost Solution)'
author: travis.hegner
layout: post
redirect_from: /2014/01/you-did-what-with-iptables-part-2-the-almost-solution/
dsq_thread_id:
  - 2149815626
categories:
  - I.T.
---
If you haven&#8217;t read [Part 1: The Problem][1], I&#8217;d suggest you go do that first.

After discovering the actual problem, I started pouring over UCM documentation and options. Different combinations of options cause the SIP trunk to behave in different ways, but there doesn&#8217;t seem to be anything related to this mystical port 4000. Finally I discover some documentation covering UCM port usage which lists TCP/4000 as a &#8220;phantom&#8221; port that UCM uses when it&#8217;s not expecting to receive any return data. Well, it appears that UDP also uses this so-called &#8220;phantom&#8221; port because that is exactly the nature of it&#8217;s use in this scenario. In most cases, this wouldn&#8217;t be an issue, but when you try to run SIP over the internet, it becomes a blatent issue because of everything described in [Part 1][1]. Even in the advanced options, there is no &#8220;revert to old behavior&#8221;.

Why do they do this? I can only speculate, but I would guess it&#8217;s to shave a handful of milliseconds off of the SDP body creation process. Since it isn&#8217;t expecting to actually receive any data they can simply shove a hard coded &#8220;phantom&#8221; port into the SDP, without actually having to consult the networking stack to find an available port to be used later. It&#8217;s a little silly, because once we actually send the RTP stream, it has to consult the network stack for an available port *anyway*, they are simply putting the delay later into the dialogue, and then sending the RTP with some unknown random high-port. This simply proves that Cisco&#8217;s view of SIP is a little box on my local network which does it&#8217;s own magic with a provider. Well, my environment doesn&#8217;t work that way. I run a pure software phone system all the way to the provider, all over the internet.

Why? Because a typical branch was taken from an $800/mo phone bill, to $50/mo. This equates to well over $100,000 per year in telephone savings across the organization. I have hundreds of endpoints across the country all sharing from a pool of dozens virtual phone lines, with hundreds of &#8220;local&#8221; phone numbers across the country. Right now, it costs a branch about $10-$11/mo per desk phone. This is with all taxes, usage, and long distance included. And it scales magically. The more phones I add to this system, the cheaper it gets per phone.

<span style="line-height: 1.5em;">Alright, so I realized that can&#8217;t solve this problem by changing the source of the problem, I&#8217;m going to have to get a little creative. Let&#8217;s do a quick re-cap of what I needed to accomplish. I needed to:</span>

  1. Force the remote end to send the RTP stream even while the caller is on hold.
  2. From the firewall, identify RTP packets that are part of an MOH stream.
  3. Do an SNAT on those identified packets to change the source port.

So, number one is easy&#8230; I&#8217;ve done it a couple of times already. Write a function in my SIP proxy which flips the &#8220;a=sendonly&#8221; attribute over to an &#8220;a=sendrecv&#8221; attribute. I&#8217;m already mangling all of the SIP packets anyway to assist in my remote branch NAT traversal. Done.

Number two seams simple at first, but then you remember that the UCM does more than MOH. And there is no way to distinguish whether the packet matching this rule is part of a voicemail, a conference, or an MOH stream. If I change the source port of the wrong packet, I&#8217;ll break things worse than they already are. The only way to identify the *type* of packet is by looking at the traffic coming in from this packet&#8217;s destination. So we have to look at the connection level of the firewall. Should be easy right? On our fancy statefull firewall, with our fancy connection tracking? Not so fast.

In theory, I can identify a music on hold stream if the provider is sending it to the destination port of 4000 right? Sure! Then I can [mark the connection][2] and manipulate the return traffic however I please! Yeah!

Wait&#8230; that didn&#8217;t work&#8230; Why? Because the [conntrack][3] module doesn&#8217;t see the two streams as the same connection. Why? Because the outbound source port doesn&#8217;t match the inbound destination port. DOH! A Catch 22. Try Again.

Well, I still needed to identify destinations (address/port combos) that were currently part of an MOH session, since that was the only way to distinguish outbound MOH packets. I had just learned about *[port][4]*[ knocking][5], so I started looking into those techniques with the *[recent][6]* module. I quickly discovered that the recent module only stores remote ip addresses and not ip/port combos, for good reason based on what it&#8217;s designed to do. Also, I can only match against the source of a packet. In this scenario, I will at some point need to match against a packet&#8217;s destination.

Then I discovered [IPSet][7]. With IPSet I can create dynamic lists of ip/port combos, based on certain criteria of a packet. I can also match against ip/port combos within a set. Source *or* destination. Sweet.

`sudo ipset create MOH hash:ip,port timeout 1<br />
sudo ipset create TWO hash:ip,port timeout 1`

So these commands create two lists to hold ip/port combos. I have to have two lists so that I can block all outbound  RTP streams from UCM, until the stream has been properly identified as either music on hold (MOH) or two-way (TWO). Since the UCM is so much closer to my firewall than the remote end, I will almost always receive those un-identified RTP packets first.

You wouldn&#8217;t expect this to be an issue, but I learned (the hard way, of course) that the outbound packets establish an entry in the conntrack module, and only the first packet of a connection is matched against the *nat* table. So if I receive an outbound packet first, it&#8217;s un-identified, and therefore not in the MOH list. Then it goes out with the default SNAT rule for my UCM. All future packets in this stream are matched and NATed the same way, even though we have since added the destination to the MOH list (upon arrival of the first *return* packet). I could find no way to clear the connection state of a matched packet from an iptables rule, forcing a re-evaluation of the nat table rules. Perhaps this could be a future enhancement. Maybe I&#8217;ll get around to a feature request one day. It would simplify this particular configuration by a lot, as you&#8217;ll soon discover.

Continuing, let&#8217;s Identify the streams, and add them to the correct set using the SET target:

`sudo iptables -t raw -N MOH-LIST<br />
sudo iptables -t raw -A PREROUTING -d 1.1.1.1/32 -i eth1 -p udp -j MOH-LIST<br />
sudo iptables -t raw -A MOH-LIST -d 1.1.1.1/32 -i eth1 -p udp -m udp ! --dport 4000 -j SET --del-set MOH src,src<br />
sudo iptables -t raw -A MOH-LIST -d 1.1.1.1/32 -i eth1 -p udp -m udp --dport 4000 -j SET --del-set TWO src,src<br />
sudo iptables -t raw -A MOH-LIST -d 1.1.1.1/32 -i eth1 -p udp -m udp ! --dport 4000 -j SET --add-set TWO src,src<br />
sudo iptables -t raw -A MOH-LIST -d 1.1.1.1/32 -i eth1 -p udp -m udp --dport 4000 -j SET --add-set MOH src,src`

<span style="line-height: 1.5em;">In (mostly) plain English, this set of commands takes the source address and port of any UDP traffic destined for port 4000 and removes that combo from the TWO list, and also stores that combo to the MOH list. For any UDP packet NOT (!) destined for port 4000, it does the opposite. The removals are necessary because in the case of a transfer to voicemail, an RTP stream is quickly changed from MOH to two way, and a lingering entry in the list (even with a one second timeout) could cause us to rewrite valid packets. Also take notice that we are doing this in the </span>*[raw][8]* table. This is because the packets we are using for identity are ultimately dropped by the forward table, as there isn&#8217;t a connection established yet.

When I originally had this identification process set up at the beginning of the forward table, the ip/port combos were never stored to the list. I will assume it&#8217;s because they were dropped before the chain was finished being processed. Now, let&#8217;s not forget to block the un-identified outbound RTP packets, and allow the identified ones through using the &#8220;set&#8221; match:

`sudo iptables -N MOH-BLOCK<br />
sudo iptables -I FORWARD -s 10.10.10.10/32 -i eth0 -o eth1 -p udp -j MOH-BLOCK<br />
sudo iptables -A MOH-BLOCK -s 10.10.10.10/32 -i eth0 -o eth1 -p udp -m set --match-set TWO dst,dst -j ACCEPT<br />
sudo iptables -A MOH-BLOCK -s 10.10.10.10/32 -i eth0 -o eth1 -p udp -m set --match-set MOH dst,dst -j ACCEPT<br />
sudo iptables -A MOH-BLOCK -s 10.10.10.10/32 -i eth0 -o eth1 -p udp -j DROP`

First we direct all outbound UDP packets to our new custom chain &#8220;MOH-BLOCK&#8221;. Here we can test whether the current packet has been identified as an MOH packet, or a two-way packet. If the packet is identified, the firewall allows the outbound packets through. Now we just have to do an SNAT for our public address and port 4000 on only our MOH packets (obviously before our default SNAT for UCM)!

`sudo iptables -I POSTROUTING -s 10.10.10.10/32 -o eth1 -m set --match-set MOH dst,dst -j SNAT --to-source 1.1.1.1:4000`

Once the outbound packets are through, a connection is established and the inbound packets are let in as well. And this works! BRAVO!

> &#8220;What&#8217;s that&#8230;?&#8221; &#8220;Oh, the second time&#8230;?&#8221; &#8220;Ah, I see&#8230; I&#8217;ll look into it.&#8221;

As luck should have it, our master plan has a flaw. If you put the caller on hold, they hear hold music. If you take them off of hold, and then put them back on within 180 seconds, they don&#8217;t. Interesting.

So I begin digging again. During the *second* time the user is on hold, I can see the RTP stream coming from the provider, and I can issue a `sudo ipset list` and verify that the remote destination is correctly placed in the MOH set. I can see the outbound RTP packets entering the firewall, but never leaving. What is it this time!?!

I happened to notice something interesting in all of this analysis. My provider, no matter how many times I re-establish the media stream to a different endpoint, no matter the reason, they always re-establish the media stream with the *same* remote address/port combination. My UCM does *not*. Each time the caller is placed on hold, UCM begins using a **new** random high-port.

Now here is where it get&#8217;s interesting (you mean you aren&#8217;t interested already? This stuff is amazingly entertaining!). A UDP &#8220;connection&#8221; as it&#8217;s seen by the state machine is simply a set of ip/port combos that identifies traffic that has come through the firewall within the last X seconds. With an UNREPLIED connection (one way traffic, no return traffic) that connection lasts 30 seconds by default. With an ASSURED connection (two way traffic from same ip/ports in both directions) lasting 180 seconds. The timer is re-set with each packet that traverses the connection.

Since we are now SNATing the port, the *new* set of ip/port addresses matches an existing connection (Really conntrack? ***Now*** you want to match my connection!?). But since the original source port from my UCM is not the same as what the connection was established with, the firewall thinks it&#8217;s a port address collision and drops the outbound RTP packets (Or, I assume that is why anyway).

Not to worry though. We use the most powerful and flexible firewall in the world! We use iptables! Just jump into the raw table again and drop a NOTRACK target onto our outbound MOH RTP stream.

`sudo iptables -t raw -A PREROUTING -s 10.10.10.10/32 -i eth0 -p udp -m set --match-set MOH dst,dst -j NOTRACK`

Alright! Just a quick packet capture to make sure everything is wor&#8230; ewww&#8230; Yeah, shoving private IP addresses out to the internet is *definitely not* going to work.

Not to fear! We can just simply do&#8230;. a&#8230;&#8230; stateless SNAT? No? Maybe the raw table has a target to&#8230;? No. What about the mangle table, that would be the perfect&#8230; Not there either? Huh.

I guess the nat table requires connection tracking. It&#8217;s for performance reasons I suppose. Since it doesn&#8217;t evaluate the nat table for every packet, it uses the information stored as part of the connection to correctly re-write the later packets as they traverse the firewall. I&#8217;m a little surprised though that there really isn&#8217;t a way to just manually flip the address and port to what I want it to be. Understandably, how could kernel developers foresee a need to do stateless nat when we have a very nice, full featured statefull nat system available. Of course, a target could be written for the mangle table to do what we need I&#8217;m sure. It is, after all, in the open source world. But that would take me a long time, because I&#8217;ve never really written a kernel module before.

So is it back to the white board on this one? Find out in <a href="http://travishegner.com/2014/01/you-did-what-with-iptables-part-3-the-actual-solution/" target="_blank">Part 3</a>!

 [1]: http://travishegner.com/2014/01/you-did-what-with-iptables-part-1-the-problem/
 [2]: http://security.maruhn.com/iptables-tutorial/x9125.html
 [3]: http://www.iptables.info/en/connection-state.html
 [4]: https://wiki.archlinux.org/index.php/Port_Knocking
 [5]: https://wiki.archlinux.org/index.php/Port_Knocking#Port_Knocking_with_iptables_only
 [6]: http://www.snowman.net/projects/ipt_recent/
 [7]: http://ipset.netfilter.org/
 [8]: http://www.iptables.info/en/structure-of-iptables.html#RAWTABLE