---
title: 'You did WHAT with iptables? (Part 4: The "Revert to old behavior" Solution)'
author: travis.hegner
layout: post
redirect_from: /2015/04/you-did-what-with-iptables-part-4-the-revert-to-old-behavior-solution/
categories:
  - I.T.
---
I was conferring with <a href="http://clinta.github.io" target="_blank">my newest colleague</a>, and explaining to him the hack job I had to do to make music on hold work. Apparently, his google-fu is better than mine, as he found this gem:

<a href="https://supportforums.cisco.com/discussion/11890741/unicast-ucm-music-hold-fails-through-cube-asr" target="_blank">https://supportforums.cisco.com/discussion/11890741/unicast-ucm-music-hold-fails-through-cube-asr</a>

Turns out, they did put in a "revert to old behavior" option. It's called "Duplex Streaming Enabled".

Needless to say, that works much better than what I was doing.