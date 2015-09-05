---
title: 'Cloudera Manager: Too Many Open Files'
author: travis.hegner
layout: post
---
Working with cloudera manager 5.4.5, I recently discovered a bug in the way they handle limits on the cloudera-scm-server service. The symptoms start with the web interface of the cloudera manager being completely unresponsive when trying to access it. SSH into the host running the service, and it's java process is consuming a lot of CPU.

Executing:

{% highlight bash %}
sudo service cloudera-scm-server restart
{% endhighlight %}

Will fix the issue, but only temporarily.

As you dig into the issue, you realize that the service is still logging to the `.log.1` file, rather than the normal `.log` file. Tailing the file will yield the error `Too Many Open Files`.

First I tried increasing the overall `fs.max-files` for the whole system with `sysctl`, but the issue still occurred. After some research, I found that the process itself was hitting it's own limits, via the `/etc/security/limits.conf` system. The system also includes a `/etc/security/limits.d/` directory for custom settings. In the directory is even a file for `cloudera-scm.conf`. In that file, there are settings for the `cloudera-scm` user which up the `nofile` soft and hard limits.

After discovering this, and analyzing `/proc/<pid>/limits` for the cloudera-scm-server service, I realized that the settings for that user were not being applied to that process. More [research][1] showed that the limits system is only applied at login through pam, and this service was being started by an init script.

I added the following `ulimit` commands in between the lines as shown:

{% highlight bash %}
echo -n "Starting $prog: "
ulimit -n 32768
ulimit -Hn 1048576
$CMF_SUDO_CMD /bin/bash -c "nohup $SERVER_SCRIPT $CMF_SERVER_ARGS" > $SERVER_OUT 2>&1 </dev/null &
{% endhighlight %}

And cloudera manager has been running well since.

[1]: http://serverfault.com/questions/359185/limits-conf-not-being-applied
