---
title: Hadoop 0.20 on Ubuntu Server 9.04 Jaunty
author: travis.hegner
layout: post
permalink: /2009/06/hadoop_020_on_ubuntu_server_904_jaunty/
dsq_thread_id:
  - 2149846028
categories:
  - Hadoop
  - I.T.
  - Tutorials
---
In the last few weeks, I have become completely fascinated by clustered computing, and solving very large complex problems with a somewhat large number of cheap computers. While researching more efficient ways to store and search through my employers internal resume database, I stumbled onto this software called [Hbase][1], which is a [column-oriented database][2], modeled after [Google BigTable][3]. To increase speed and reliability, Hbase runs best on a distributed file system provided by the [Hadoop Core][4] project (HDFS).

Documentation is plentiful for older versions of Hadoop, installed on older versions of Ubuntu, but there were significant changes in both Hadoop 0.20 and Ubuntu Server 9.04 that created a few stumbling points for my test installations. That being said, this tutorial will be on how to install a hadoop 0.20 cluster on any number of Ubuntu Server Jaunty 9.04 machines.

In light of the current economic times, we have quite a few extra desktops sitting around here at work, so I took the liberty of installing ubuntu server on a few (6) of them to build a test cluster. I relied heavily on the [Hadoop online documentation][5] as well as the tutorials by [Michael G. Noll][6] on [single-node clusters][7] and [multi-node clusters][8]. The only problem I have with my cluster at this point is not having enough data to process with it!

Before you begin, make sure you have a dedicated switch for you cluster, an ubuntu server 9.04 install cd, and all the machines ready for an OS that you want to put into your cluster. I highly recommend running static IP&#8217;s on all the machines in your cluster, as well as an internal authoritative DNS server. You could even [install BIND][9] on the &#8220;Namenode&#8221; for a dedicated solution. If not, you&#8217;ll have to manually keep every hosts file up to date on every node. I&#8217;ve found that Hadoop can be very finicky about DNS name resolution, so you have to be careful about what&#8217;s in your hosts file, as well as your nsswitch.conf file. You definately don&#8217;t want your machine to grab a dhcp address at any point in time during the install, as it will write your hosts file and potentially your resolv.conf file differently than you need it. The minimum requirements for you cluster machines is really dependant on the problem you want to solve, or the amount of data you want to analyze. Each node in my cluster has about 512 MB of RAM, 30-80 GB of hard disk space, and a 3.0 GHz HT P4 processor. I am also only running on a 10/100 Mb switch. If you want to do something serious, then you may want to look at some better hardware, and definately a 1Gb switch, but if your just fooling around, your dusty 686&#8217;s, and a simple 10/100 Mb switch should do the trick. Your nodes will need internet access, so be sure that is available on your cluster network as well.

First, you&#8217;ll want to install ubuntu server on each of your nodes. I usually just set the user during the install to hadoop, to avoid having to create this user later. Make sure after the install you run a dist-upgrade,

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo apt-get update&lt;br />
sudo apt-get dist-upgrade&lt;br />
</code>
  </p>
</blockquote>

and then install sun-java6-jdk.

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo apt-get install sun-java6-jdk&lt;br />
</code>
  </p>
</blockquote>

Next, you&#8217;ll have to select a single machine to be your &#8220;NameNode&#8221; and  
your &#8220;JobTracker&#8221;. This is basically the Master node in the cluster,  
and will do a little bit more work than the rest of the nodes, so  
choose your most powerful machine. If you are building a large cluster  
for a production system, you&#8217;ll want your NameNode and JobTracker to be on  
separate, dedicated boxes, and You&#8217;ll also want to make sure you have a  
&#8220;SecondaryNameNode&#8221; for redundancy. Since my cluster is small, and only  
for experimentation, I only have a single master node and all of the  
master processes combined on to it. It even runs the slave  
processes as well to help store the data, and spread the work load  
around.

The next step is to setup ssh key authentication from the master to  
each slave node. It is imperitive that the following commands be  
executed as the hadoop user on the master node.

<blockquote class="code">
  <p>
    <code>&lt;br />
ssh-keygen -t rsa -P ""&lt;br />
ssh-copy-id hadoop@&lt;master-node&gt;&lt;br />
ssh-copy-id hadoop@&lt;slave-node&gt;&lt;br />
... (repeat for each slave) ...&lt;br />
</code>
  </p>
</blockquote>

This will create a key for your hadoop user on your master node, and  
give that user password-less access to itself and each of the slaves.  
While copying the keys it will ask to store the fingerprint, answer yes  
each time, so that it doesn&#8217;t interfere with the master&#8217;s commands  
later. Hadoop uses ssh to start and stop the slave processes. Repeat these steps on any other &#8220;master&#8221; machine if you have opted to have a secondary name node and/or separate job tracker machines. You can get a bit more advanced and find the same key on the first master, and distribute it to all the other masters, instead of generating a new key for each master, but you&#8217;ll still have to connect to each slave and each other master at least once manually to store the fingerprint.

[Download hadoop core][10]. I typically create a directory on the root called hadoop,

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo mkdir /hadoop&lt;br />
</code>
  </p>
</blockquote>

and expand the tar.gz we just downloaded into it. Then, change the permissions on the directory to be owned by our hadoop user and group:

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo chown -R hadoop:hadoop /hadoop&lt;br />
</code>
  </p>
</blockquote>

I usually then sym-link &#8220;core&#8221; to the &#8220;hadoop-0.20.0&#8221; directory, so it&#8217;s a little easier to reference.

<blockquote class="code">
  <p>
    <code>&lt;br />
ln -s /hadoop/hadoop-0.20.0 /hadoop/core&lt;br />
</code>
  </p>
</blockquote>

Remember that all of your nodes will have to have the same version of hadoop installed, and in the same location on each node, so you&#8217;ll be repeating these steps for every node in the cluster. Make sure the following directories exist in the /hadoop directory as well: hdfs/name, and hdfs/data. 

<blockquote class="code">
  <p>
    <code>&lt;br />
mkdir /hadoop/hdfs/name && mkdir /hadoop/hdfs/data&lt;br />
</code>
  </p>
</blockquote>

My intention is to use the /hadoop directory as a top level holding place when I start experimenting with Hbase and/or any other hadoop based projects. The code base for any project can be stored in that directory to keep things organized.

A short cut is available using rsync to get this set up on all of our slave nodes. First, log in to each slave, create the /hadoop directory, modify permissions (command above), then run

<blockquote class="code">
  <p>
    <code>&lt;br />
rsync -r --progress hadoop@&lt;master-node&gt;:/hadoop/* /hadoop/&lt;br />ln -s /hadoop/hadoop-0.20-0/ /hadoop/core&lt;br />
</code>
  </p>
</blockquote>

This should copy our entire set-up to the node without having to go through unzipping and all the subsequent steps from that. You can do this after configuring hadoop so that you don&#8217;t have to configure each slave independently, but only do this if you don&#8217;t plan to have any custom settings per node. Please also do not do this if you&#8217;ve already attempted to start hadoop.

So with our install in place on all of our nodes, it&#8217;s time to begin configuring hadoop. On the master node, go to /hadoop/core/conf, and open the core-site.xml file for editing.

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /hadoop/core/conf && pico core-site.xml&lt;br />
</code>
  </p>
</blockquote>

Between the <configuration> and </configuration> tags, add the following:

<blockquote class="code">
  <p>
    <code>&lt;br />
&lt;property&gt;&lt;br />&nbsp; &lt;name&gt;fs.default.name&lt;/name&gt;&lt;br />&nbsp; &lt;value&gt;hdfs://$master-node$/&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />
</code>
  </p>
</blockquote>

Next, open the hdfs-site.xml, and add the following properties:

<blockquote class="code">
  <p>
    <code>&lt;br />
&lt;property&gt;&lt;br />&nbsp; &lt;name&gt;dfs.name.dir&lt;/name&gt;&lt;br />&nbsp; &lt;value&gt;/hadoop/hdfs/name&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />&lt;property&gt;&lt;br />&nbsp; &lt;name&gt;dfs.data.dir&lt;/name&gt;&lt;br />&nbsp; &lt;value&gt;/hadoop/hdfs/data&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />
</code>
  </p>
</blockquote>

Then, open mapred-site.xml and add the following:

<blockquote class="code">
  <p>
    <code>&lt;br />
&lt;property&gt;&lt;br />&nbsp; &lt;name&gt;mapred.job.tracker&lt;/name&gt;&lt;br />&nbsp; &lt;value&gt;$master-node$:54311&lt;/value&gt;&lt;br />&lt;/property&gt;&lt;br />
</code>
  </p>
</blockquote>

Please remember to replace $master-node$ in each of the above files with the actual dns name of your master node, and leave out the $&#8217;s, that is just so you recognize that it must be your custom name.

Next, you have to add a couple of settings to the hadoop-env.sh

<blockquote class="code">
  <p>
    <code>&lt;br />
export JAVA_HOME=/usr/lib/jvm/java-6-sun&lt;br />
export HADOOP_OPTS=-Djava.net.preferIPv4Stack=true&lt;br />
</code>
  </p>
</blockquote>

If you&#8217;ve followed some of the older tutorials for hadoop on ubuntu, namely those by Michael Noll, then you&#8217;ve realized that you can&#8217;t (yet) disable IPv6 in 9.04. The newer kernels recognize some optional paramters to disable it, but 9.04&#8217;s current kernel does not recognize those options. The second line we add to the hadoop-env.sh forces hadoop (through a java option) to use our IPv4 address, instead of the IPv6 one. For some reason, hadoop will only listen on IPv6 when it&#8217;s available. The NameNode process ties into this as well because it will grab it&#8217;s listening address from the hosts file based on the machines name. You must verify that your hosts file has your master node&#8217;s dns name listed next to the intended network address, and not 127.0.0.1, or some other address.

It may not be necessary, but for good measure I even modified my /etc/nsswitch.conf so that the &#8220;hosts:&#8221; entry only checks &#8220;files&#8221; then &#8220;dns&#8221;. The other entries (mdns) cause hostnames to be re-written to <host-name>.local, and during some troubleshooting, I worried that the re-written host names may affect hadoop.

The above configurations should all also be made on every slave, with the exception of the dfs.name.dir property. This property is only used by the namenode, so it does not need to be configured on any of the slaves, although I think a slave will safely ignore it.

Now, we need to modify the &#8220;slaves&#8221; file. This file is a list of all the machines in the cluster who should be running the &#8220;DataNode&#8221; and/or the &#8220;TaskTracker&#8221; processes. Please use the actual dns name of the master server and not &#8220;localhost&#8221; as is in there by default. Since we used the dns name earlier in the process, it will hang when it tries to ssh to &#8220;localhost&#8221; because we never accepted and saved the fingerprint for that host. Localhost will work if you save the fingerprint, but I prefer to keep things somewhat consistent. If you have separated your NameNode from your JobTracker, then you&#8217;ll need to have the same list on each master machine.

In our test cluster, we have no use for the SecondaryNameNode, so I removed all entries from the &#8220;masters&#8221; file. If you will be needing redundancy, then you&#8217;ll definitely want to look into the SecondaryNameNode Process, and how to set it up. Any machine in the cluster that should run the SecondaryNameNode, should be listed in the masters file.

Assuming you have everything configured properly, then from the master server, in the /hadoop/core directory, you can run:

<blockquote class="code">
  <p>
    <code>&lt;br />
bin/hadoop namenode -format&lt;br />
bin/start-dfs.sh&lt;br />
bin/start-mapred.sh&lt;br />
</code>
  </p>
</blockquote>

The first command should only be run once to initialize hdfs, and should never be run while the cluster is running. The second two commands will start the NameNode/DataNodes and the JobTracker/TaskTrackers respectively for every machine in the cluster. Once the cluster has had a few minutes to register itself to the master, you can point your browser to http://<master-node>:50070/ for the current cluster health, and lists of slave nodes. You can also point to http://<master-node>:50030/ for lists of jobs and their current status.

The start-hdfs.sh, stop-hdfs.sh, start-mapred.sh, and stop-mapred.sh scripts can all be used to quickly stop and start the entire cluster, just always be certain that the slaves file is up to date. If you have separated your JobTracker from your NameNode, then you must also execute those commands separately. The cluster-wide start/stop commands should only be executed from the master of that service. To manage individual slave nodes independently of the cluster, you can use bin/hadoop-daemon.sh. This script allows you to dynamically add and remove individual machines from the cluster without affecting the rest of the cluster.

<blockquote class="code">
  <p>
    <code>&lt;br />
Usage: hadoop-daemon.sh [--config &lt;conf-dir&gt;] [--hosts hostlistfile] (start|stop) &lt;hadoop-command&gt; &lt;args...&gt;&lt;br />
</code>
  </p>
</blockquote>

After completing my cluster, I decided to do some benchmarking to see what we can actually use this thing for. I stole about 1.6 GB worth of raw log files from our web server, and copied them into hdfs:

<blockquote class="code">
  <p>
    <code>&lt;br />
bin/hadoop dfs -copyFromLocal temp/* logs&lt;br />
</code>
  </p>
</blockquote>

Watching the cluster health page, you can see how it balances the storage load equally among all of the nodes. Once the copy was complete, I ran the provided wordcount job as an example:

<blockquote class="code">
  <p>
    <code>&lt;br />
bin/hadoop jar hadoop-0.20.0-examples.jar logs logs-out&lt;br />
</code>
  </p>
</blockquote>

This breaks the logs up into individual words, and then counts the number of occurrences of each word and writes it to an output file. To see how the sheer number of nodes affects the processing time of this job, I re-ran this job on my cluster with from only one node, all the way up to six. My results are as follows:

1 node:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 16m59.997s  
2 nodes:&nbsp;&nbsp;&nbsp;&nbsp; 8m34.724s  
3 nodes:&nbsp;&nbsp;&nbsp;&nbsp; 5m49.434s  
4 nodes:&nbsp;&nbsp;&nbsp;&nbsp; 4m33.927s  
5 nodes:&nbsp;&nbsp;&nbsp;&nbsp; 3m29.834s  
6 nodes:&nbsp;&nbsp;&nbsp;&nbsp; 2m57.472s

As you can see, there is a sort of nuclear decay with adding nodes verses the amount of time it takes to process. If you have researched [the beowulf project][11] at all, they talk a lot about parrallel computing theory, and explain that only so much work can be done in parrallel, and that as you get to an infinite number of nodes, the parrallel portion of the job would be done nearly instantaneously, but there always will be certain steps that can not be skipped, and must be processed serially. Not to mention the added overhead that happens as you add nodes to the cluster.

Now imagine that I had to process 16 GB of apache logs&#8230; One node would have taken roughly 170 minutes, while 6 nodes would have taken roughly 30 minutes. When you start to look at extremely large quantities of data, you can quickly see the benifit to having several machines working on a single problem.

If you are interested, you should check out the &#8220;[powered by][12]&#8221; section of the hadoop website, to see how this type of computing is being used in the real world.

Until Next Time&#8230;

 [1]: http://hadoop.apache.org/hbase/
 [2]: http://en.wikipedia.org/wiki/Column-oriented_DBMS
 [3]: http://labs.google.com/papers/bigtable.html
 [4]: http://hadoop.apache.org/core/
 [5]: http://hadoop.apache.org/core/docs/r0.20.0/cluster_setup.html
 [6]: http://www.michael-noll.com/wiki/Main_Page
 [7]: http://www.michael-noll.com/wiki/Running_Hadoop_On_Ubuntu_Linux_%28Single-Node_Cluster%29
 [8]: http://www.michael-noll.com/wiki/Running_Hadoop_On_Ubuntu_Linux_%28Multi-Node_Cluster%29
 [9]: http://www.travishegner.com/2009/06/authoritative-dns.html
 [10]: http://hadoop.apache.org/core/releases.html#22+April%2C+2009%3A+release+0.20.0+available
 [11]: http://www.beowulf.org/
 [12]: http://wiki.apache.org/hadoop/PoweredBy