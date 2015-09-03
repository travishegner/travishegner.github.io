---
title: Installing Software on Old Ubuntu Installs
author: travis.hegner
layout: post
redirect_from: /2012/03/installing-software-on-old-ubuntu-installs/
dsq_thread_id:
  - 2149842087
categories:
  - I.T.
  - Tutorials
  - Ubuntu
---
Have an old Ubuntu server or desktop and can't access the repositories? Getting messages like:

<pre>Err http://us.archive.ubuntu.com karmic-updates/main Packages
  404  Not Found [IP: 91.189.92.181 80]</pre>

And:

<pre>W: Failed to fetch http://security.ubuntu.com/ubuntu/dists/karmic-security/main/binary-i386/Packages.gz  404  Not Found [IP: 91.189.92.167 80]</pre>

Here are a couple of sed one-liners to fix it for you.

<pre>sudo sed -i 's/us.archive.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list
sudo sed -i 's/security.ubuntu.com/old-releases.ubuntu.com/g' /etc/apt/sources.list</pre>

Obviously, adjust the original mirror (us.archive.ubuntu.com) to whichever mirror you usually use if necessary.

Then, update and install your package:

<pre>sudo apt-get update
sudo apt-get install &lt;your-package&gt;</pre>

Enjoy!