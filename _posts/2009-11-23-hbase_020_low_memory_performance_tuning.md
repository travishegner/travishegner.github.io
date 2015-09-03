---
title: 'Hbase 0.20: Low Memory Performance Tuning'
author: travis.hegner
layout: post
redirect_from: /2009/11/hbase_020_low_memory_performance_tuning/
dsq_thread_id:
  - 2149845355
categories:
  - Hbase
  - I.T.
  - Tutorials
---
Hey Everybody! 

<p style="text-align: justify">
  As promised, though a little later than I'd originally hoped, here is a quick blurb on what I've done to make my fully distributed Hbase instance run with a little bit more stability.
</p>

{% highlight xml %}
<property>
<name>hbase.hregion.max.filesize</name>
<value>134217728</value>
<description>Default: 268435456</description>
</property>
{% endhighlight %}

<p style="text-align: justify">
  This setting is the maximum file size of a region in any given Hbase table. Setting this to a lower value will inevitably increase the number of regions served, but it seems to help prevent time-outs when dealing with desktop grade hardware. It's a little easier for the system to move around and work with slightly smaller files.
</p>

{% highlight xml %}
<property>
<name>hbase.hstore.compactionThreshold</name>
<value>2</value>
<description>Default: 3</description>
</property>
{% endhighlight %}

<p style="text-align: justify">
  This setting controls how often a <em>mem store</em> file will be compacted into a full region file. The mem store file is created when a mem store is flushed to disk. Setting this to a lower value, increases the number of compactions you have, but it decreases the amount of time it takes to compact. During my testing, my clients were timing out during compactions, so this setting helped to prevent that.
</p>

{% highlight xml %}
<property>
<name>hbase.hregion.memstore.flush.size</name>
<value>33554432</value>
<description>Default: 67108864</description>
</property>
{% endhighlight %}

<p style="text-align: justify">
  This is the mem store mentioned earlier. Once the store reaches the configured size, it is flushed to disk. Setting this value lower gives you your biggest bang for saving memory, as it forces data to disk sooner, and uses less memory overall.
</p>

<p style="text-align: justify">
  Please experiment with these settings on your own accord. Your results may differ, but for me, these settings made the difference that allowed my very large import to complete without a single failure.
</p>
