---
title: Authoritative DNS
author: travis.hegner
layout: post
redirect_from: /2009/06/authoritative_dns/
dsq_thread_id:
  - 2149846177
categories:
  - I.T.
  - Tutorials
---
As much as I love exploring and learning the newer protocols and technologies like SIP, and Enterprise Virtualization, there is something to be said for those standards and protocols that the internet is built from on and designed around. It is quite refreshing to work with a protocol that is not under constant revision and modification, since the software that runs it is rock solid, stable, and just runs until you tell it not to.

If you haven&#8217;t caught on, or are new to my site, I prefer running everything I can on [ubuntu server edition][1]. I&#8217;ve found it to be stable, fast, reliable, lightweight, and still powerful enough to handle anything from large database back-end systems, to lightweight web hosting systems, and even to, you guessed it, authoritative DNS servers. I won&#8217;t suggest an ubuntu server version here, since it is nearly the same procedure on every version I have used (I&#8217;ve been using ubuntu server since 2006.)

First, you need a server to run it on. If you have a virtual infrastructure already handy, DNS is ideal as a virtualized system because it uses very, very little system resources, unless you are hosting DNS for some EXTREMELY busy domains. If you don&#8217;t have a virtual infrastructure, or any handy hardware, you can comfortably share your DNS services on your web or email server. Just bear in mind that this server will have to be publicly available, so choose a machine that is safe for that purpose.

After several difficult and dissapointing conversations with first level technical support from various ISPs, I decided a few years ago that I would manage my own authoritative DNS solutions. I was pretty much fed up when the last ISP I had hosting my DNS told me that there was no such thing as a [TXT Record][2].

So once you have a server ready to host your DNS, you&#8217;ll want to install [BIND][3]:

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo apt-get install bind9&lt;br />
</code>
  </p>
</blockquote>

Once bind is installed, you can immediatly begin to configure it. My standard procedure is to create the directory where I store all of my &#8220;zone files&#8221;. 

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo mkdir /etc/bind/zones&lt;br />
</code>
  </p>
</blockquote>

A zone file is a file which describes your domain, it&#8217;s hosts, and how other users of the internet should use the information you provide. Next we edit the file /etc/bind/named.conf.options. In here, you can specify some of the many, many options of bind. I typically will set up &#8216;allow-recursion&#8217;, and &#8216;version&#8217; directives here.&nbsp; If you are setting up an internal DNS server, or would like bind to do recursive queries for your internal clients, then you would want to include your internal address range:

<blockquote class="code">
  <p>
    <code>&lt;br />
allow-recursion { 192.168.0.0/16 };&lt;br />
</code>
  </p>
</blockquote>

If this server will be relying on itself for DNS lookups, then you&#8217;ll want to make sure that the localhost can recurse as well:

<blockquote class="code">
  <p>
    <code>&lt;br />
allow-recursion { localhost; 192.168.0.0/16; };&lt;br />
</code>
  </p>
</blockquote>

There are some other tricks you can find at:  
<http://www.zytrax.com/books/dns/ch7/queries.html>  
[http://www.zytrax.com/books/dns/ch7/address\_match\_list.html][4]  
Just be careful and put some thought into who you will, and will not allow to do recursive queries.

The version directive is recommended by <http://www.cyberciti.biz/faq/hide-bind9-dns-sever-version/> to be set to something generic like &#8220;BIND&#8221;:

<blockquote class="code">
  <p>
    <code>&lt;br />
version "BIND";&lt;br />
</code>
  </p>
</blockquote>

Otherwise it is possible for an attacker to obtain your version number, and exploit some yet undiscovered vulnerabilities in your version. Some of the newer versions of BIND instituted a new set of logging directives which I typically set, otherwise your syslog will be littered with tons of error messages from servers in the world that don&#8217;t follow the DNS RFC standards properly:

<blockquote class="code">
  <p>
    <code>&lt;br />
logging {&lt;br />&nbsp; category lame-servers { null; };&lt;br />&nbsp; category edns-disabled { null; };&lt;br />};&lt;br />
</code>
  </p>
</blockquote>

Next we have to modify the file /etc/bind/named.conf.local, which holds all of our yet to be configured &#8220;zones&#8221;. Zones are usually domains, but can be sub-domains, or even in-addr.arpa domains for doing reverse DNS. Once inside the file, add a zone like so:

<blockquote class="code">
  <p>
    <code>&lt;br />
zone "myexampledomain.com" {&lt;br />&nbsp; type master;&lt;br />&nbsp; file "/etc/bind/zones/db.myexampledomain.com";&lt;br />};&lt;br />
</code>
  </p>
</blockquote>

This tells BIND that it is authoritative for the domain &#8220;myexampledomain.com&#8221;, and that the zone file can be found at &#8220;/etc/bind/zones/db.myexampledomain.com&#8221;. The zone file name and location is my personal preference, you can store it wherever, and name it whatever you&#8217;d like. Once you configure a zone file, you can restart the service (/etc/init.d/bind9 restart) and begin testing your newly created DNS server. I won&#8217;t go into detail about zone files, but a simple google search for &#8220;bind zone file&#8221; will get you going. Here is an example zone file:

<blockquote class="code">
  <p>
    <code>&lt;br />
$TTL 86400&lt;br />myexampledomain.com.&nbsp; IN&nbsp; SOA&nbsp; ns1.myexampledomain.com. hostmaster.myexampledomain.com. (&lt;br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 2007100301&nbsp;&nbsp;&nbsp;&nbsp; ; Serial&lt;br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 3600&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ; Refresh 3 hours&lt;br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 3600&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ; Retry 1 hour&lt;br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 604800&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ; Expire 1 week&lt;br />&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 86400&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ); Negative TTL: Minimum 5 minutes&lt;/p>
&lt;p>@&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; NS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ns1.myexampledomain.com.&lt;br />@&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; NS&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ns2.myexampledomain.com.&lt;br />@&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; A&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 192.168.0.1 ;web site ip&lt;br />ns1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; A&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 192.168.0.2 ;bind server ip&lt;br />www&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; IN&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; CNAME&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; myexampledomain.com.&lt;br />
</code>
  </p>
</blockquote>

With the zone file in place, restart your BIND server and we can begin testing. Type nslookup and hit enter. Here you can issue dns queries against your default server or specify a server:

<blockquote class="code">
  <p>
    <code>&lt;br />
server localhost&lt;br />
</code>
  </p>
</blockquote>

Now query for your domain and you should get the following response:

<blockquote class="code">
  <p>
    <code>&lt;br />
&gt; myexampledomain.com&lt;br />Server:&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp; localhost&lt;br />Address:&nbsp;&nbsp;&nbsp; 127.0.0.1#53&lt;/p>
&lt;p>Name:&nbsp;&nbsp;&nbsp; myexampledomain.com&lt;br />Address: 192.168.0.1&lt;br />
</code>
  </p>
</blockquote>

If you see something like this, then your configuration is successful. If not, check your logs, and your configuration files, and try again.

 [1]: http://www.ubuntu.com/products/whatisubuntu/serveredition
 [2]: http://www.rfc-editor.org/rfc/rfc1464.txt
 [3]: http://en.wikipedia.org/wiki/BIND
 [4]: http://www.zytrax.com/books/dns/ch7/address_match_list.html