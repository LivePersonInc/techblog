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

**troubleshooting** Troubleshooting changes drastically.  If by now you were used to troubleshooting your servers by checking out graphs in your favorite monitoring tool, and in more difficult cases accessing your servers and running various commands to check its health, web pages or `jmx` on specific hosts to check its status then now you basically have a few things you need to change.  First and foremost the machine names, you are not working with machines but you need to locate your containers via your orchestrator, be it for example `kubernetes` or `fleet`.  `kubernetes` services take care of exposing ports which are used to access your server as a business logic unit, but what about accessing ports on your specific app node? if a server which runs `jconsole` is out of the cluster you would need to expose the ports to it.  What about running utilities on the container to check it's status.  Let's take for example `tcpdump`.    

It is most likely that you are not going to have troubleshooting tools such as `lsof` and `tcpdump` in your container.



**App and deployment types** You must be having many flavours of servers.  You must be having http style servers, but do you have also storm topologies? hadoop jobs? spark? spark streaming? `vert.x`? databases? which flavours? That means you should cosinder which server types are your candidates for `docker` adoption.  Usually you wish to start with plain servers, `web/app` servers as opposed to converting any kind of `databases` or servers which are not in frequent development and deployment.  Also in most cases you would need to scale up and down your `web/app` servers, so again this makes them a good candidate, In addition those server's are almost completely in your control, so again it makes sense to start with these servers.  Among these you would first want to `POC` a few or a single non critical service, it's best that your first `POC` contains a server which if it's down no harm done.  do a full `docker` adotpion on it and test it.  Once you feel comfort and you have formed procedures and templates for `docker` adoption you may continue gradually to servers with higher criticallity and with stricter `SLA`.  Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.

**Service discovery** In this item we'll refer to `kubernetes` as the `containers cluster manager`.  If you go through `kubernetes proxy` you can get access to the cluster by accessing any node of the kubernetes cluste on the ports which the `service` exposes.  However, you don't want to be dependant on the nodes themselfs.  So we can add another layer of abstraction in manner of an `external load balancer`.  Here we have 2 ways, we can query the `kubernetes api server` for the updated ports and ips of our service or we can define them as static in the service.  In anyway our client contacts a permanent `DNS` we can setup on our external load balancer and the load balancer will point to the service on the kubernetes cluster.   If you don't do that you are tightly coupled with the kubernetes cluster which you don't want to.  You want your system tests to run on a rather permanent set of locations on `QA` env (which can simply be `QA namespace`) and you can setup a different `DNS` for every env (namespace).

**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.


