---
title: Adding Hbase 0.20 to Our Hadoop Cluster
author: travis.hegner
layout: post
permalink: /2009/08/adding_hbase_to_our_cluster/
dsq_thread_id:
  - 2149845783
categories:
  - Hbase
  - I.T.
  - Tutorials
---
OK, OK, I know what you&#8217;re thinking. &#8220;Where has this guy been with some more of this hadoop goodness that we just can&#8217;t get enough of?&#8221; Well, I am (finally) back, and you&#8217;ll be happy to learn what I brought with me: A whole mess more Hadoop and Hbase knowledge that I can&#8217;t wait to show you!

Nevertheless, the time has finally come. The brilliant developers over at [Hbase][1] have made a release candidate for version 0.20 available for [download][2]. I believe that the changes have slowed enough to document the installation procedures a bit. The link is for RC1, but be on the lookout for RC2, which should be available very soon. I will document the procedures for both downloading the pre-built package, and installing from svn.

First, I am going to recommend you follow my [hadoop tutorial][3], because the power of Hbase only really begins to shine when you have it installed fully distributed on a cluster.

Please make a note that in order to get Hbase running with some real stability, I was forced to upgrade my cluster a bit. First, I moved my master onto a virtual machine, so that it was off of the &#8220;workhorse&#8221; nodes, and on a more redundant set of &#8220;hardware&#8221;. Effectively, this gave me another slave node to do some work, so now I&#8217;m up to a 7 node cluster! I also upgraded the ram on each node to 1GB. Still low, but much more stable since doing so.

Log into your master node, and navigate to the /hadoop directory:

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /hadoop&lt;br />
</code>
  </p>
</blockquote>

Make sure you have subversion and ant installed:

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo apt-get install subversion ant&lt;br />
</code>
  </p>
</blockquote>

Check out the code from the apache subversion repository:

<blockquote class="code">
  <p>
    <code>&lt;br />
svn co http://svn.apache.org/repos/asf/hadoop/hbase/branches/0.20 hbase-svn&lt;br />
</code>
  </p>
</blockquote>

This will always be the latest and greatest code for the 0.20 branch. (The trunk is now working towards 0.21). If you would like a &#8220;release&#8221; that is not under development, then have a look at http://svn.apache.org/repos/asf/hadoop/hbase/tags/ and you can pick from previously released &#8220;snapshots&#8221; of the code. Just adjust the &#8220;svn co&#8221; command accordingly. If you have downloaded a pre-built package, then simply extract the contents to the /hadoop/hbase-<version>/ directory. Assuming all has gone well, we are ready to configure our hbase install. Navigate to the hbase/conf directory that you are currently using:

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /hadoop/hbase-svn/conf&lt;br />
</code>
  </p>
</blockquote>

Add the following line to your hbase-env.sh file:

<blockquote class="code">
  <p>
    <code>&lt;br />
export JAVA_HOME=/usr/lib/jvm/java-6-sun&lt;br />
</code>
  </p>
</blockquote>

In the same file change this line:

<blockquote class="code">
  <p>
    <code>&lt;br />
export HBASE_OPTS="-XX:+HeapDumpOnOutOfMemoryError -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode&lt;br />
</code>
  </p>
</blockquote>

to look like:

<blockquote class="code">
  <p>
    <code>&lt;br />
export HBASE_OPTS="-XX:+HeapDumpOnOutOfMemoryError -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode -Djava.net.preferIPv4Stack=true"&lt;br />
</code>
  </p>
</blockquote>

Now, modify your &#8220;regionservers&#8221; file to list all of the machines you want to host regions. Think of an Hbase region as a small chunk of the data in your database. The more regionservers you have, the more data you can reliably serve. In my cluster, the regionservers are the same nodes as all of my datanodes, and all of my tasktrackers. So, essentially, the &#8220;regionservers&#8221; file should be identical to your &#8220;slaves&#8221; file from the hadoop tutorial.

Next, modify the hbase-site.xml file. The settings in this file over-write those in hbase-default.xml, so if you want to see a list of available settings to configure, then study that file, but only make changes to your hbase-site.xml. Add the following settings to hbase-site.xml:

<blockquote class="code">
  <p>
    <code>&lt;br />
&lt;property&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;name&gt;hbase.rootdir&lt;/name&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;value&gt;hdfs://$master$/hbase&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />&lt;property&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;name&gt;hbase.cluster.distributed&lt;/name&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;value&gt;true&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />&lt;property&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;name&gt;hbase.zookeeper.quorum&lt;/name&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;value&gt;$slave1$,$slave2$,$slave3$&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />&lt;property&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;name&gt;hbase.zookeeper.property.dataDir&lt;/name&gt;&lt;br />&nbsp;&nbsp;&nbsp; &lt;value&gt;/hadoop/zookeeper/data&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />
</code>
  </p>
</blockquote>

Please remember to replace $master$ and $slaveX$ with your master and slave host names respectively. You may have read that Hbase 0.20 now requires zookeeper, but fear not, the above configuration directives allow hbase to completely manage zookeper on it&#8217;s own, you never have to mess with it. Now, it is typically recommended to always run zookeeper on dedicated zookeeper only servers. If you are running a small cluster, then this is hardly efficient, because you want as many nodes &#8220;working&#8221; as possible. While I can&#8217;t give you recommendations of the maximum cluster size you can have before requiring dedicated zk nodes, I can tell you that my 6 slave nodes run datanode, tasktracker, regionserver, and zookeeper without too much of a problem. I would imagine that if you have over 10 nodes in your cluster, then you shouldn&#8217;t have a problem dedicating a few for zookeeper. They also recommend (maybe even require) that zookeeper runs on an odd number of machines. I don&#8217;t completely understand how zookeeper works, but basically as long as you still have more than half of your &#8220;quorum&#8221; in tact, then your cluster won&#8217;t fail. In essence, if your zk quorum has 7 nodes, you can lose 3 nodes without any adverse affects, a 35 node quorum could theoretically lose 17 nodes, and still operate. I think basically zookeeper is used to keep track of the locations of regions, so your quorum will notify any clients, and fellow regionservers where to find the data they are looking for. If zk becomes overloaded, then your regionservers can time out and crash, and potentially lose data if they haven&#8217;t flushed to disk yet. So make sure you have enough horsepower for your application. In my cluster, the hbase.zookeeper.quorum directive is simply a comma separated list of all of my slave nodes, including my master. If you have an odd number of slaves (even number counting your master), then just leave the master out of the list. If you have more than ten slaves, then consider dedicating 3 of them to zookeeper if you have problems with regionservers timing out. The logs will tell you if that is the case.

Now, if you are building from svn, then run the following command from your hbase-svn directory:

<blockquote class="code">
  <p>
    <code>&lt;br />
ant&lt;br />
</code>
  </p>
</blockquote>

This command is like &#8220;make&#8221;, but for java. It will compile the java code into .class binary files, .jar files, and even copy our config to hbase-svn/build. This new build directory is basically the same as a pre-built downloaded package, so I always link to it from /hadoop/hbase:

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /hadoop && ln -s hbase-svn/build hbase&lt;br />
</code>
  </p>
</blockquote>

If you are not building from svn, then simply link /hadoop/hbase to your hbase-<version> directory:

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /hadoop && ln -s hbase-&lt;version&gt;&lt;version> hbase&lt;br />
&lt;/version></code>
  </p>
</blockquote>

Simply remember to replace hbase-<version> with your actual directory name. Please also remember that only hbase-0.20.X will run on our hadoop cluster.

Next we have copy our hbase install to all of the nodes that will be running a regionserver. I wrote a handy script to do this for me, that way as I build, and rebuild hbase, I can simply run my &#8216;deploy-hbase.sh&#8217; script to spread it around my cluster:

<blockquote class="code">
  <p>
    <code>&lt;br />
#!/bin/bash&lt;/p>
&lt;p>rsync -r --progress --delete hbase-svn slave2:/hadoop/&lt;br />rsync -r --progress --delete hbase-svn slave3:/hadoop/&lt;br />rsync -r --progress --delete hbase-svn slave4:/hadoop/&lt;br />rsync -r --progress --delete hbase-svn slave5:/hadoop/&lt;br />rsync -r --progress --delete hbase-svn slave6:/hadoop/&lt;br />rsync -r --progress --delete hbase-svn slave7:/hadoop/&lt;br />
</code>
  </p>
</blockquote>

Add or remove lines to correspond to your slaves, and put the proper names/directories in as necessary. After hbase has been copied to each of the slaves, we must log into each slave for some final steps. Every node in a cluster has to have the code-base installed in the same location so we have to link each of our nodes /hadoop/hbase directory to the same location that we linked to on the master. For each of our zk nodes, we must also create a data directory, and id file:

<blockquote class="code">
  <p>
    <code>&lt;br />
mkdir -p /hadoop/zookeeper/data && echo 'X' &gt; /hadoop/zookeeper/data/myid&lt;br />
</code>
  </p>
</blockquote>

It is imperative that you replace the &#8216;X&#8217; with &#8216;0&#8217;, on the first node in your quorum, &#8216;1&#8217; on the second, &#8216;2&#8217; on the third and so on. This file allows the node to identify itself in the zk quorum.

Once all that per node work is done, you can finally start your hbase instance. From the master /hadoop/hbase directory run:

<blockquote class="code">
  <p>
    <code>&lt;br />
bin/start-hbase.sh&lt;br />
</code>
  </p>
</blockquote>

If all goes well, you can navigate your browser to http://<master>:60010/ to see some stats about the running hbase and zk instances.

Keep checking back, as there is plenty more to come:  
* Tuning Hadoop/Hbase for low memory clusters (like mine, truly commodity hardware!)  
* A full map/reduce example job utilizing hbase as a source and destination for data, written natively in java (a first for me).  
* A document search algorithm utilizing your map/reduce output to find exactly what we are looking for, with ranking included.

Thanks for reading, come back soon!

 [1]: http://hadoop.apache.org/hbase/
 [2]: http://people.apache.org/%7Estack/hbase-0.20.0-candidate-1/
 [3]: http://www.travishegner.com/2009/06/hadoop-020-on-ubuntu-server-904-jaunty.html