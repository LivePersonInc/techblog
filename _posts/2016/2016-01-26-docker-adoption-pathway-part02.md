---
layout: post
title:  "Docker adoption pathway - Part 2"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: true
---
# Docker adoption pathway - Part 2

In part 1 we went through the motivation to adopt `containerized deployments` some of it's effect on deployment tools.  To remind, having a new service in a new company containerized is a lot differnet from having an existing or even new service containerized when you already have deployment tools and monitoring in place.  You have to do a shift.  Let's check which aspects need to be reexamined while doing this move, in this part we are going to see through troublehooting in containerized environments versus standard ones, the different app types we might have, which ones are good candidates for containerization and service discovery. 

**troubleshooting** Troubleshooting changes drastically.  You might have been used to troubleshooting your servers by checking out graphs in your favorite monitoring tool.  In cases where it required you would access your servers and run various commands to check its health (`uptime`, `ps`, `lsof`, `tcpdump`, `dmesg`, `sar`, and more troubleshooting assistant commands).  Many times there are web pages on the `app` instances themselves and-or `jmx` exposed on jvm based `apps` to check the status of the app and the machine.  When you have your app `containerized` things change.  First and foremost the machine names, you are not working with machines but you need to locate your containers via your `orchestrator`, be it `kubernetes` or `fleet` or other orchestration tools.  With regards to having the web pages or jmx exposed `kubernetes` services take care of exposing ports which are used to access your server as a business logic unit, but what about accessing ports on your specific app node? if a server which runs `jconsole` is out of the cluster you would need to expose the ports to it.  What about running utilities on the container to check it's status.  Let's take for example `tcpdump`.  We are going to use a `containerized cassandra` as an example and see that we have access to it's network packets with `tcpdump`.
    
Let's first start up a `dockerized cassandra container`.

```bash
$ docker run --name containerized-cassandra -d cassandra 
91523e6a1e34f52e89993ae75821633a92b2528c5e0f551983a9518f7044d286
```

Let's start a second cassandra container which would result in having it in a cluster with the first:
```bash
docker run --name containerized-cssandra2 -d -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' containerized-cassandra)" cassandra:latest
```

Let's verify both of them are running:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
34950087146a        cassandra:latest    "/docker-entrypoint.s"   12 minutes ago        Up 4 seconds        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   containerized-cssandra2
91523e6a1e34        cassandra           "/docker-entrypoint.s"   10 minutes ago        Up 12 seconds       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   containerized-cassandra
```

Access the container with shell:

```bash
$ docker exec -it containerized-cassandra bash
root@91523e6a1e34:/#
```

and execute our favorite network analysis tool - `tcpdump`

```bash
# tcpdump
bash: tcpdump: command not found
```

for the sake of the case let's use another very common troubleshooting unix command `lsof`.

```bash
# lsof
bash: lsof: command not found
```

So we don't have any `tcpdump` nor `lsof` in our `container`.  While it is expected it actually means to us - troubleshooting is different - in containerized environments.

So if we wish to use one of the above commands what should be our path, there are multiple ones, lets check our options and choose what we think suits best this problem.  Our options:

1. Troubleshoot the container `externally`.  Run the commands from the `node` itself.
1. Use another container perhaps a `troubleshooting container` to run the commands.
1. Install the `troubleshooting` commands inside our `app` container.
1. Use new abstractions for troubleshooting for example docker has `docker top` command so you can run externally to the container `docker top containerized-cassandra`.
1. Use 3rd party tools which allow greater visibility into your containers.

As always we first see if we have standards, as we don't have any `docker tcpdump` obviously then our second best option we have opted for is to run to stat a new container running `tcpdump` and use it to analyze the network traffic.

```bash
$ docker run --net=host --rm corfr/tcpdump -iveth9258f66 port 7000
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth9258f66, link-type EN10MB (Ethernet), capture size 65535 bytes
22:02:43.063875 IP 172.17.0.2.53094 > 172.17.0.3.afs3-fileserver: Flags [P.], seq 2575582772:2575582908, ack 3499689297, win 229, options [nop,nop,TS val 2370991 ecr 2370807], length 136
```

**App and deployment types** You must be having many flavours of servers.  the most common are `http` style services; but, do you have also `storm topologies`, `samza cluster`? `hadoop` jobs? `spark streaming`? `spark jobs?` `vert.x` server? `databases servers` obviously but which flavours? That means you should consider which server types are your candidates for `container` adoption.  Usually you wish to start with plain servers, `web/app` servers, stateless as possible, as opposed to converting any kind of `databases` or `job` like processes.  Also, this would give you an immediate hube benefit for autoscaling your `web/app` services, so again this makes them good candidates, In addition those server's are almost completely in your control, as opposed to servers which already have cluster management built in.  So again it makes sense to start with these servers.  Among these the path to gradually convert is that it's possible to first `POC` a few or a single non critical service, one which if its down not too much harm done, it's best that your first `POC` contains a server which if it's down no harm done.  do a full `docker` adotpion on it and test it.  Once you feel comfort and you have formed procedures and templates for `docker` adoption you may continue gradually to servers with higher criticallity and with stricter `SLA`.  Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.

**Service discovery** In this item we'll refer to `kubernetes` as the `containers cluster manager`.  If you go through `kubernetes proxy` you can get access to the cluster by accessing any node of the kubernetes cluste on the ports which the `service` exposes.  However, you don't want to be dependant on the nodes themselfs.  So we can add another layer of abstraction in manner of an `external load balancer`.  Here we have 2 ways, we can query the `kubernetes api server` for the updated ports and ips of our service or we can define them as static in the service.  In anyway our client contacts a permanent `DNS` we can setup on our external load balancer and the load balancer will point to the service on the kubernetes cluster.   If you don't do that you are tightly coupled with the kubernetes cluster which you don't want to.  You want your system tests to run on a rather permanent set of locations on `QA` env (which can simply be `QA namespace`) and you can setup a different `DNS` for every env (namespace).

**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.


