---
layout: post
title:  "Docker adoption pathway - Part 2"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: false
---
# Docker adoption pathway - Part 2

**Fast and Slow lane**  Your teams are already used to doing their tasks, development - providing tested artifacts to production and production team for their proper installation on production environments.  Once you break the current methodology you have to consider the occasions where you need to perform a quick update on production.  Now that your app with OS conf is already bundled in an image, would you still be able to perform a quick fix when needed?  When dealing with artifacts and their deployment on production you should always have two lanes.  Fast and Slow.  By all means the slow lane is used always.  Always except for the time you have no other choice and you need to make the fix on production immediately.  As much as you should avoid this you must have this option available, even theoretically so that when time comes is available for you.  With containerized deployments the fast lane should be checkout the image in production, perform the fix, commit and apply it.  It is quick enough to enable you to discard the option of login into a container and doing the updates, you can do that, but you need to recall that this container would not nessesaraly live long enough to take on the change provided in the slow lane.  The slow lane includes updating the source code in development env, creating a new image which goes through the cycle of all tests and providing this image back to production.

<img src="https://docs.google.com/drawings/d/1QS6fZGaJuRIVfSG5bx5Sjq6E6C5xZHdyrluc3ycs7GM/pub?w=386&amp;h=252">

**App and deployment types** You must be having many flavours of servers.  You must be having http style servers, but do you have also storm topologies? hadoop jobs? spark? spark streaming? `vert.x`? databases? which flavours? Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.

**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.