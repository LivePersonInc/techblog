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

**Service discovery** In this item we'll refer to `kubernetes` as the `containers cluster manager`.  Obviously from within the cluster you can access services with their `DNS` name.  However on machines which are outside of the cluster, be it your own `dev machine` or another node, you would not be able by default to access the internal cluster `kubernetes services`.  You need to solve this, access from outside the cluster inside the cluster.  There are multiple ways to achieve that and you need to choose the one that bests suit you.

**App and deployment types** You must be having many flavours of servers.  You must be having http style servers, but do you have also storm topologies? hadoop jobs? spark? spark streaming? `vert.x`? databases? which flavours? That means you should cosinder which server types are your candidates for `docker` adoption.  Usually you wish to start with plain servers, `web/app` servers as opposed to converting any kind of `databases` or servers which are not in frequent development and deployment.  Also in most cases you would need to scale up and down your `web/app` servers, so again this makes them a good candidate, In addition those server's are almost completely in your control, so again it makes sense to start with these servers.  Among these you would first want to `POC` a few or a single non critical service, it's best that your first `POC` contains a server which if it's down no harm done.  do a full `docker` adotpion on it and test it.  Once you feel comfort and you have formed procedures and templates for `docker` adoption you may continue gradually to servers with higher criticallity and with stricter `SLA`.  Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.

**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.



