---
title: PostgreSQL 8.3 Warm Stand-by Replication
author: travis.hegner
layout: post
redirect_from: /2009/06/postgresql_83_warm_stand-by_replication/
dsq_thread_id:
  - 2149846364
categories:
  - I.T.
  - Tutorials
---
In an effort to make things a bit more redundant at my work place, I have implemented [PostgreSQL 8.3 warm stand-by replication][1], as outlined in the [pgsql 8.3 manual][2].The manual leaves the possibilities for implementation pretty wide open, which is good for making things scalable, and applying the process in many different ways, but it is bad if you are looking for more detail as to how to set it up. In this tutorial, I will outline my exact methods for implementing warm stand-by for PostgreSQL, running on Ubuntu Server 9.04.

The basic jist of postgresql's warm stand-by system, is based on the "[continuous archiving][3]" ability of pgsql. It utilizes this feature to do "log shipping" of the write ahead log (WAL) files, where the logs are then retrieved by a stand-by server in "continuous recovery" mode. The stand-by server remains in continuous recovery until a given condition has been met, at which point the server comes up into normal operation.

First, you must start with two "as close to the same as possible" servers, as recommended by the postgresql manual. In my case, I have a virtual pgsql server in our primary location, and another virtual pgsql server in our disaster recovery location. Both servers run ubuntu server 9.04 jaunty 32-bit, with postgresql installed from the repositories. PostgreSQL also requires that the two servers be the same architecture (32 or 64 bit).

Once both servers are running, you must create a location to store the archived WAL files. It is highly recommended to have the archive storage located remotely from the primary server, that way, in the event of a primary server failure, you have the latest and greatest WALs for your stand-by server to import. The archive must be readable, and writeable from both the primary and secondary servers. For my installation, I created a NFS share on the stand-by server itself, and then the stand-by server imports the WAL files locally. Just be sure that the archive on the stand-by server is owned by "postgres:postgres", and that the world permissions are "0" because all data written to your database will be visible here as it is written. Remember to take precautions, security is your number one concern at all times.

With the remote archive mounted locally, we must configure pgsql for continuous archiving. Edit /etc/postgresql/8.3/main/postgresql.conf and set the "archive\_mode" directive to "on". For my purposes, I set the "archive\_command" to "test -f /psql\_archive/mounted && test ! -f /psql\_archive/%f && rsync -a %p /psql\_archive/%f". I created an empty file called "mounted" on the NFS share, so that I could verify that the share was actually mounted, and prevent writing WAL files to the local place-holder archive directory. So our archive\_command basically first checks to see if the share is mounted, If it is, then it checks to see if the current WAL already exists, if it doesn't, then it writes the file with rsync. The -a parameter of rsync is important because it first copies the data to a place-holder file name, and then changes the name of the destination file only after all the data was successfully copied. This prevents the stand-by server from trying to load a partial log file.

The archive\_timeout directive will require some careful thought on your part. Postgresql will only archive a WAL when it reaches 16MB worth of writes, or when the data becomes older than the "archive\_timeout" in seconds. Unfortunately, the WAL file is ALWAYS 16MB, whether we have written that much data or not. So setting this value too low, will exponentially increase the amount of bandwidth required to transmit your WALs. If the stand-by server is on the LAN, this is less of an issue, but over the WAN, you'll need to really be careful how often the the archive times out. Please also remember that if you have 16MB of data written to the database before your time-out, it will ship the log at that point in time, so if you are replicating over the WAN, plan your bandwidth capacity according to the requirements of your database's write activity. For me, the archive_timeout is only every 10 minutes, which is safe because we don't have rapidly changing data, and losing 10 minutes worth of data in the event of a failure would not be catastrophic. This only averages out to around&nbsp; 220kbps of continuous bandwidth utilization.

Your main concern with WAN replication during idle database times, is that you do not want the next WAL file attempting to transmit before the previous one has finished. This will cause a chain reaction which will lead to your secondary database falling further and further behind, without an opportunity to catch up. If you happen to run into this during peak periods of activity, it shouldn't hurt, so long as it's able to catch up within a time frame that is acceptable to you.

After setting these parameters, and restarting the postgresql service (/etc/init.d/postgresql-8.3 restart) you can watch the archive directory, and you should start seeing WAL files being transmitted. We have to be careful, because if we run in this state too long, we will run our archive out of disk space.

Now we can begin configuring our stand-by server for continous recovery mode. It is my preference to create a new file in /etc/postgresql/8.3/main/recovery.conf and symbolic link to it, but you can optionally create the file in the final destination directory which I'll mention later.

The continuous recovery portion would be quite a bit more complicated if it weren't for "pg\_standby". Luckily, pg\_standby is installed and compiled by default in ubuntu server 9.04, but you must create a symbolic link to pg_wrapper from /usr/bin.

<blockquote class="code">
  <p>
    <code>&lt;br />
sudo ln -s /usr/share/postgresql-common/pg_wrapper /usr/bin/pg_standby&lt;br />
</code>
  </p>
</blockquote>

Once that is complete, just open your recovery.conf file and add the line (please disregard the line wrap around, this should be a single line in the file):

<blockquote class="code">
  <p>
    <code>&lt;br />
restore_command = 'pg_standby -d -l -r 3 -s 60 -t /psql_archive/role.master /psql_archive %f %p %r 2&gt;&gt;/var/log/postgresql/pg_standby.log'&lt;br />
</code>
  </p>
</blockquote>

This line basically tells postgresql to continue recovery forever until the file /psql_archive/role.master exists. In order to make my fail-over in the event of a disaster a bit more seamless, I have written a custom cron job to create this file automatically, based on an internal DNS change. To put it as simply as possible, I have a generic CNAME "postgres" in my internal DNS, which all of my postgresql clients point to for service. Under normal circumstances, the CNAME points to "pg-primary". In the event of a fail-over, I will change the CNAME so that it points to "pg-secondary". Adding the following line to your /etc/crontab file on your stand-by machine will automatically create your "trigger file" after making your DNS change:

<blockquote class="code">
  <p>
    <code>&lt;br />
* *     * * *   postgres    test -n "`nslookup postgres | grep pg-secondary`" && touch /psql_archive/role.master &gt;/dev/null 2&gt;&1&lt;br />
</code>
  </p>
</blockquote>

I still wouldn't go as far as calling this a "hot standby" just yet, but it does simplify the process of failing over.

So to actually get this thing up and running, we must first take a base backup of the data directory of our primary server. It is neither necessary, nor recommended to shut the server down for the base backup. From a SQL prompt, issue the query:

<blockquote class="code">
  <p>
    <code>&lt;br />
select pg_start_backup('base_backup');&lt;br />
</code>
  </p>
</blockquote>

After that, create a tar archive of the entire data directory. On ubuntu server 9.04, the postgresql data directory is located at /var/lib/postgresql/8.3/main. To do this, issue the command:

<blockquote class="code">
  <p>
    <code>&lt;br />
tar -czf base_backup.tar.gz /var/lib/postgresql/8.3/main&lt;br />
</code>
  </p>
</blockquote>

Once the tar file is made, connect back to the database and issue the query:

<blockquote class="code">
  <p>
    <code>&lt;br />
select pg_stop_backup();&lt;br />
</code>
  </p>
</blockquote>

Copy the resulting file to your stand-by server. Once there, shut down postgresql on the stand-by server, and extract the contents of the tar.gz archive, overwriting the existing data directory. Navigate to the data directory:

<blockquote class="code">
  <p>
    <code>&lt;br />
cd /var/lib/postgresql/8.3/main&lt;br />
</code>
  </p>
</blockquote>

Then create a symbolic link to /etc/postgresql/8.3/main/recovery.conf, if you chose to make it there, or create your recovery.conf file here.

<blockquote class="code">
  <p>
    <code>&lt;br />
ln -s /etc/postgresql/8.3/main/recovery.conf&lt;br />
</code>
  </p>
</blockquote>

Assuming everything is set up right, you should be able to start the postgresql service (/etc/init.d/postgresql-8.3 start). Monitoring the archive directory, you should see the WAL log files start to be consumed and removed every minute or so after one appears.

Unfortunately, there is no way to verify that the data is actually being replicated and recovered into the stand-by server, because during "recovery mode" the database is not operational. To test that everything is working properly, you have to manually create the trigger file:

<blockquote class="code">
  <p>
    <code>&lt;br />
touch /psql_archive/role.master&lt;br />
</code>
  </p>
</blockquote>

This should trigger the recovery process to complete, and the server should come on-line automatically. To verify, check the pgsql data directory (/usr/lib/postgresql/8.3/main) and see if the recovery.conf file has been re-named to recovery.complete. If so, then connect to the secondary database and check for the existance of some known changes since the base backup. If all looks well, then you must repeat the steps above starting from creating the original base backup, in order to get the stand-by server into recovery mode again. Simply re-naming the recovery.complete back to recovery.conf WILL NOT WORK because once the server has been started, it creates a unique 'instance' or 'timeline' and it will not start a recovery process unless the 'instance' of the WAL files match that of the recovering server. Hence the necesity of re-doing the base backup.

I also recommend testing the automatic trigger file by changing the DNS with a NON PRODUCTION set of DNS names, to make sure that it works properly. What works for me, may not work for you.

There are a number of options for recovery to status quo after a complete fail-over scenario. You could reconfigure each of the two servers so that the secondary becomes the primary. You could use the pg\_dump and pg\_restore tools to migrate your data back to the original primary, and follow the steps in this article from the base backup again to get replication going.

Another method may be to simply do a shutdown the original secondary server, then make a tar of it's data directory, extract it over top of the original primary server data directory (once repaired of course), then follow the steps to start up the secondary server in recovery mode again. I have not tested that method of recovery, but theoretically it should work as both instances, or timelines should be the same. Please try this last method at your own risk.

Disclamer: Please note that this set up may not work for any/every environment, and I can not and will not be held responsible if the use of this information results in data loss or damage. It is your responsibility to make backups and use precautions while working in a production environment with production data. The information here is provided as-is, and without warranty. That being said, I will be willing to help you out if you run in to a snag, or my directions aren't quite clear enough.

Further Reading, related articles, and sources:  
<http://www.postgresql.org/docs/8.3/static/backup-file.html>  
<http://www.postgresql.org/docs/8.3/static/continuous-archiving.html>  
<http://www.postgresql.org/docs/8.3/static/warm-standby.html>  
<http://www.postgresql.org/docs/8.3/static/pgstandby.html>

 [1]: http://www.postgresql.org/docs/8.3/static/warm-standby.html
 [2]: http://www.postgresql.org/docs/8.3/static/index.html
 [3]: http://www.postgresql.org/docs/8.3/static/continuous-archiving.html