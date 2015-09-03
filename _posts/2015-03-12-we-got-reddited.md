---
title: We Got Reddited
author: travis.hegner
layout: post
permalink: /2015/03/we-got-reddited/
dsq_thread_id:
  - 3589792354
categories:
  - I.T.
---
So, someone on <a href="http://www.reddit.com/r/firstworldanarchists/comments/2ys5r3/nsfw/" target="_blank">reddit</a> thought it would be rather funny to post a link to this image on our server at work:

<img src="/images/hardhatsmall.jpg" alt="hardhatsmall" />

&nbsp;

It was posted under the heading &#8220;NSFW&#8221; (Not Safe For Work). If you aren&#8217;t aware, that typically denotes a post to something inappropriate in nature. It&#8217;s work safety gear, so it&#8217;s pretty funny. One of the <a href="http://www.reddit.com/r/firstworldanarchists/comments/2ys5r3/nsfw/cpcn224" target="_blank">comments</a> on the thread suggested that: &#8220;Their sysadmin is going to be very confused.&#8221; Indeed, we were:

<img src="/images/reddit_hit.png" alt="reddit_hit" />

&nbsp;

The original image was rather large in nature, so we were moving a lot of outbound traffic (a lot for us anyway). So, to mitigate, I resized the image, and did an http 301redirect back to a custom (and very slim) html page:

<img src="/images/reddit_landing.png" alt="reddit_landing" />

We&#8217;ve never had that much traffic before, and we appreciate the opportunity to see what it&#8217;s like&#8230; even for a silly reason.
