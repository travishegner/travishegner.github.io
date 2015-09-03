---
title: We Got Reddited
author: travis.hegner
layout: post
redirect_from: /2015/03/we-got-reddited/
dsq_thread_id:
  - 3589792354
categories:
  - I.T.
---
So, someone on [reddit](http://www.reddit.com/r/firstworldanarchists/comments/2ys5r3/nsfw/) thought it would be rather funny to post a link to this image on our server at work:

![hardhatsmall](/images/hardhatsmall.jpg)

It was posted under the heading *NSFW* (Not Safe For Work). If you aren't aware, that typically denotes a post to something inappropriate in nature. It's work safety gear, so it's pretty funny. One of the [comments](http://www.reddit.com/r/firstworldanarchists/comments/2ys5r3/nsfw/cpcn224) on the thread suggested that: "Their sysadmin is going to be very confused." Indeed, we were:

![reddit_hit](/images/reddit_hit.png)

The original image was rather large in nature, so we were moving a lot of outbound traffic (a lot for us anyway). So, to mitigate, I resized the image, and did an http 301 redirect back to a custom (and very slim) html page:

![reddit_landing](/images/reddit_landing.png)

We've never had that much traffic before, and we appreciate the opportunity to see what it's like, even for a silly reason.
