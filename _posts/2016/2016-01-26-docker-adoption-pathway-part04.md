---
layout: post
title:  "Docker adoption pathway - Part 4"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: false
---
# Docker adoption pathway - Part 3

**configuration** 

A `container` as it's name promises is a self container application to get deployed and run.  This raises a question, if its self contained how would configuration which by it's nature is an external dynamic input to the container be encapsulated and shipped into the container to change its behaviour?  As always there is no one magical bullet to achieve that.  In this post we will scan our options discussing when should we use each option.

First off let's discuss which configuration layers we have, in general we can divide configuration layers to:

1. app-config - your specific application configuration it could be the local folder to store cache, connection pool size, etc.
1. app-infra-config - usually you app relies on another server, such as `nodejs` / `nginx` / ... this configuration refers to layer just below your app such as `nginx.conf`
1. orchestrator config - `kubernetes` / `docker swarm` configuration, in the older `puppet` world this could also be `puppet` configurations, templates.
1. infra/os config - your actual operation system configuration, example `ulimit`
1. network config - the network architecture of your cluster including `VIP` and load balancing.

for minimal configuration like one parameter pass it as argument, for a few more pass them as environment variable, always prefer to have convention for example if you use some cassandra host simply refer to DNS CASSANDRA in dev would be the right one in prod the right one.  For more complex configuration consider storing them in some kind of database or in volume files.  At first you see you have `yaml` files to define your `service` and `rc`.  But what if you need replication factor of one kind in one env and in another env a different `replication factor`?  You either end up with multiple duplicated `yaml` files or you `hack` something out.  The current recommended solution is to use `jinja2` as a templating tool so when you pass the `yaml` files for `kubernetes` to digest you have a few more `parameter` files *all under revision control ofcourse*.  You use the `diff` files as input to the `yaml` files so that you can run kubernetes on each env with its own specialized files. 

In addition where do you store those `yaml` configuration file? answer - your version control repository usually git repository.  You check them out on the machine you run `kubectl` on and run on them.

In addition what about your `app` configuration files? you might need also a different configuration file per `env`.  Ofcourse you should prefer `convetion over configuration` and if you still have a small set of configurations to pass to your app you can pass them through environment parameters or you can fetch them from a network storage like a `database`.  But what happens in the cases you still have local configuration files to configure for your app?  You should find some technique to `plant` those configuration files when your container is installed.  most probably via `volume` mounting.  You can mount the volume as a [gitRepo-volume](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/volumes.md#gitrepo), this means you store your configuration and manage their revisions in git repository.  Note that if you still have multiple configurations per multiple envs you need to utilize [jinja-templating][jinja-link]
 
resources:
see links


# Docker adoption pathway - Part 4


and as you can discover from the API we get many other metrics such as memory.  This we can work with.

**Example 2 Running tcpdump in containerized environments**

Let's work with an **actual example** to see the changing troubleshooting in front of our eyes.  In our case we need to run a network analyzer tool - `tcpdump` in order to analyze the packets sent in between `containers`.  In this example we are going to use a `containerized cassandra` and go through the process of using `tcpdump` to analyze its communication, we would just try to run `tcpdump`.
 
 Note: we are going to start by **naively** trying to run the tools as we did prior `containers era` and then refactor our process until it matches the `containers` spirit.

Let's first start up first cassandra container: 

```bash
docker run --name containerized-cassandra -d cassandra 
91523e6a1e34f52e89993ae75821633a92b2528c5e0f551983a9518f7044d286
```

So we now have our first cassandra container up.  Let's start a second cassandra container and `seed` it with the first cassandra container so that both instances are aware of each other's existence:

```bash
docker run --name containerized-cssandra2 -d -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' containerized-cassandra)" cassandra:latest
```

We have started the second cassandra container and passed to it `CASSANDRA_SEEDS` environment variable which includes the first's cassandra `ip`.  Let's verify both of them are running:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
34950087146a        cassandra:latest    "/docker-entrypoint.s"   12 minutes ago        Up 4 seconds        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   containerized-cssandra2
91523e6a1e34        cassandra           "/docker-entrypoint.s"   10 minutes ago        Up 12 seconds       7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   containerized-cassandra
```

Both are running.  Let's access the first container with `shell` access:

```bash
$ docker exec -it containerized-cassandra bash
root@91523e6a1e34:/#
```

Let's start executing troubleshooting utilities start with `uptime`

```bash
# date
Wed Feb 24 09:55:49 UTC 2016
root@bab3991fd904:/# uptime
 09:55:52 up  1:07,  0 users,  load average: 0.64, 1.07, 1.18
```

And it has presented the time and number of users with relation to the container. 

Let's move on to executing more troubleshooting commands - and execute our favorite network analysis tool - `tcpdump`

```bash
# tcpdump
bash: tcpdump: command not found
```

well, it didn't work, would other troubleshooting tools work? `lsof` perhaps?

```bash
# lsof
bash: lsof: command not found
```

So we don't have any `tcpdump` nor `lsof` in our `container`.  Well, I said we were starting naive.  Actually, the way to think of `containers` is more of as **processes** than a full blown servers what was true for `VM's` isn't true for containers, at least today.  Our problem was that that the container looked like a server, it walked like a server and it quaked like a server, but it's not an actual full blown server **nor** a `lightweight vm` and this is why we got mislead.  We should expect not to have these tools preinstalled in our container and you are most likely to be in this situation quiet a lot for other containers which are not in your control.  What it actually means to us is that in containerized environments troubleshooting is different which is expected but could be misleading at times.

So, if we wish to use one of the above troubleshooting tools what should be our path? well, there are multiple ones, lets check our options and choose what we think suits best this problem.  Our options with regards to **`tcpdump`** (before filtering and prioritizing):

1. Troubleshoot the container **`externally`**.  Run `tcpdump` from the `node` itself.
1. Use another container perhaps a **`troubleshooting container`** with pre-installed `tcpdump` just for such cases.
1. **Install** the `troubleshooting` commands inside our `app` container.
1. Use new container troubleshooting abstractions, for example, docker has **`docker top`** command so you can run externally to the container `docker top containerized-cassandra`.
1. Use **3rd party** tools which allow greater visibility into your containers.

As always we first see if we have standards, we obviously don't have any `docker tcpdump` then the second best option we have opted for is to run a new container running itself `tcpdump` and use it to analyze the network traffic.  We could have a collection of such troubleshooting containers for such cases, we could then trigger them when needed, we do not clutter our host we have them in containers.

```bash
$ docker run --net=host --rm corfr/tcpdump -iveth9258f66 port 7000
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth9258f66, link-type EN10MB (Ethernet), capture size 65535 bytes
22:02:43.063875 IP 172.17.0.2.53094 > 172.17.0.3.afss3-fileserver: Flags [P.], seq 2575582772:2575582908, ack 3499689297, win 229, options [nop,nop,TS val 2370991 ecr 2370807], length 136
```

And this is it, we should just listen to the traffic from the outside, our container is only meant to run our app, need another tool running? be it troubleshooting or any other? run it in another container.

**Example 3 Connecting to JMX in containerized environments**

Let's connect to one of our containers `JMX`.  We start `jvisualvm` and we try to connect to its jmx, we are going to use [jmxterm](http://wiki.cyclopsgroup.org/jmxterm/) in order to do that from the `commandline`
`cassandra` exposes jmx, according to it's documentation it's on port: [7199 - Cassandra JMX monitoring port.](https://docs.datastax.com/en/cassandra/2.0/cassandra/security/secureFireWall_r.html)

```bash
$ java -jar ~/Downloads/jmxterm-1.0-alpha-4-uber.jar  --url localhost:7199
```

which results with:

```bash
java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.ServiceUnavailableException [Root exception is java.rmi.ConnectException: Connection refused to host: localhost; nested exception is: 
	java.net.ConnectException: Connection refused]
```

let's check with `telnet` if we have any connection to our `cassandra` via `jmx port` which is `7199`:

```bash
telnet localhost 7199
:~/tmp/java-docker$ telnet localhost 7199
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
```

No connection.  If we look at `cassanra's Dockerfile` we see that it does `expose` port `7199`:

```Dockerfile
# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
EXPOSE 7000 7001 7199 9042 9160
```
 
However, it is not going to be exposed to an `unlinked container` or to our `visualvm/jconsole` from outside the container environment.  For that purpose when we `run` the `cassandra container` we are going to use the `-p` option in order to `expose` that port also on the `node` itself.  So our `docker run` command would be as following:

```bash
docker run --name containerized-cassandra -p 7199:7199 -d cassandra
```

So that when we run `docker ps` we can verify that that port was exposed:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                       NAMES
11b499d74c4e        cassandra           "/docker-entrypoint.s"   25 seconds ago      Up 24 seconds       7000-7001/tcp, 9042/tcp, 0.0.0.0:7199->7199/tcp, 9160/tcp   containerized-cassandra
```

You see that the only port exposed is the jmx port: `0.0.0.0:7199->7199/tcp`

So now if we check if that port is exposed we see that it is:

```bash
$ telnet localhost 7199
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

now we try again with `jmxterm`:

```bash
java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: error during JRMP connection establishment; nested exception is: 
	java.net.SocketException: Connection reset]
```

We have `connection reset`.  So before we got `connection refused` because no port was exposed at all and now `connection reset`.  Let's login to the image and try to understa why this happens:

```bash
$ docker exec -it containerized-cassandra bash
```

Let's find the startup script to cassnadra and see where it configures `jmx`:

```bash
root@104f281f5f09:/# grep cassandra /etc/init.d/cassandra
# Provides:          cassandra
NAME=cassandra
CONFDIR=/etc/cassandra
CASSANDRA_HOME=/usr/share/cassandra
[ -e /usr/share/cassandra/apache-cassandra.jar ] || exit 0
[ -e /etc/cassandra/cassandra.yaml ] || exit 0
[ -e /etc/cassandra/cassandra-env.sh ] || exit 0
CMD_PATT="Dcassandra-pidfile=.*cassandra\.pid"
```

we see that casssandra has a `cassandra-env.sh` it's used to configure the cassandra instance started lets look at it and see how it configures the `jmx`:

```bash
root@104f281f5f09:/# more /etc/cassandra/cassandra-env.sh
```

and we find this interesting piece:

```bash
if [ "$LOCAL_JMX" = "yes" ]; then
  JVM_OPTS="$JVM_OPTS -Dcassandra.jmx.local.port=$JMX_PORT -XX:+DisableExplicitGC"
else
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.port=$JMX_PORT"
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.rmi.port=$JMX_PORT"
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.ssl=false"
```

so if we do `ps -ef | grep cassandra` we find that we have `-Dcassandra.jmx.local.port=7199` which means we are using the `local.port` and not setting the `com.sun.management.jmxremote.rmi.port` which means that `LOCAL_JMX` is `yes`.  therefore we are going to change `LOCAL_JMX` to `no` by means of passing an environment variable to our cassandra docker image.

```bash
$ docker run --rm --name containerized-cassandra -p 7199:7199 -e LOCAL_JMX='no' cassandra
```

we get `Error: Password file not found: /etc/cassandra/jmxremote.password`, let's see if the folder `/etc/cassandra` already exists in the container, because if it's not we are free to mount it:
  
```bash
$ docker exec -it containerized-cassandra bash

root@33acd467a75b:/# ls -la /etc/cassandra 
total 100
drwxrwxrwx  5 cassandra cassandra  4096 Feb  7 07:29 .
drwxr-xr-x 75 root      root       4096 Feb  7 07:29 ..
-rw-r--r--  1 cassandra cassandra 10200 Jan  7 21:41 cassandra-env.sh
-rw-r--r--  1 cassandra cassandra  1200 Jan  7 21:41 cassandra-rackdc.properties
-rw-r--r--  1 cassandra cassandra  1358 Jan  7 21:41 cassandra-topology.properties
```

There are multiple paths here to take:

1. Contribute to the original cassandra `image` with an update specifying where the `jmxremote.password` exists.
1. Mount the `/etc/cassandra` externally with a `volume` to allow you to edit these configurations externally.
1. Extend the `cassandra` image and add to it your own `jmxremote.password` file.
1. Connect the jmx passwords to your `LDAP` service.

While the `LDAP` service is our favorite option, and while this is a repeating theme, local changes are moving to the network, for this example we are going to create our own image and add to it the `jmxremote.password` file.

We create a new `jmxremote.password`:

```bash
controlRole somepass
```

Our `Dockerfile` would look as following:

```bash
FROM cassandra
COPY jmxremote.password /etc/cassandra/
```

build it:

```bash
$ docker build -t "mycassandra" .

Sending build context to Docker daemon 3.072 kB
Step 1 : FROM cassandra
 ---> 368a7c448425
Step 2 : COPY jmxremote.password /etc/cassandra
 ---> 73a3a757047e
Removing intermediate container ce8cf319d160
Successfully built 73a3a757047e
```

Run it with `LOCAL_JMX=no`

```bash
$ docker run --rm --name containerized-cassandra -p 7199:7199 -e LOCAL_JMX='no' cassandra
tomerb@TOMERBD_LAP:~/lpdev/social/techblog/_posts/2016/docker-adoption-pathway-part02$ docker run --rm --name containerized-cassandra -p 7199:7199 -e LOCAL_JMX='no' mycassandra
Error: Password file read access must be restricted: /etc/cassandra/jmxremote.password
```

So we restrict the file access, `Dockerfile`:

```bash
FROM cassandra
COPY jmxremote.password /etc/cassandra/
RUN chmod -R 600 /etc/cassandra/jmxremote.password
```

We build again the image:

```bash
$ docker build -t "mycassandra" .Sending build context to Docker daemon 3.072 kB
Step 1 : FROM cassandra
 ---> 368a7c448425
Step 2 : COPY jmxremote.password /etc/cassandra/
 ---> Using cache
 ---> fa43c7c67638
Step 3 : RUN chmod -R 600 /etc/cassandra/jmxremote.password
 ---> Using cache
 ---> 7f02e0aaf316
Successfully built 7f02e0aaf316
```

and run:

```bash
$ docker run --rm --name containerized-cassandra -p 7199:7199 -e LOCAL_JMX='no' mycassandra
INFO  17:11:49 Starting listening for CQL clients on /0.0.0.0:9042 (unencrypted)...
INFO  17:11:49 Not starting RPC server as requested. Use JMX (StorageService->startRPCServer()) or nodetool (enablethrift) to start it
INFO  17:11:50 Scheduling approximate time-check task with a precision of 10 milliseconds
INFO  17:11:50 Created default superuser role 'cassandra'
```

and now login to jmx with the `user` we defined:

```bash
$ java -jar ~/Downloads/jmxterm-1.0-alpha-4-uber.jar --user controlRole --url localhost:7199
Authentication password: ********
Welcome to JMX terminal. Type "help" for available commands.
$>
```

SUCCESS! we have connected to our `containerized cassandra's JMX`.


**Which apps to first convert** You must be having many `app` flavours with the most common `http` and `websocket` based services; but, do you also have `samza cluster`? `storm topologies`? `hadoop` jobs? `spark streaming`? `spark jobs?` `vert.x` servers? `nodejs`? `nginx`? `databases servers`, `standalone` apps?.  That means you should consider which server types are your candidates for `container` adoption or at least which are the first.  While we may claim that any app can and should be `containerized` each such app `containerization` requires an effort, with limited resources you should choose the ones your organization would gain the most out of their containerization.  

In most cases you wish to start with plain services servers; `http` and `websocket` services are in this category, `stateless` as possible, no `disk cache`, no `affinity`, and the least changing configuration.  That is as opposed to converting any kind of `databases` or `job` like processes.  Also, this would give you an immediate benefit of autoscaling your `web/app` services which usually do require that .  In addition those server's behaviour are mostly managed by you, there is no external cluster scheduler for them, as opposed to servers which already have scheduling built in, so again it makes sense to start with these services.  

Among these the path to gradually convert is that it's possible to first `POC` a few or a single non critical service, one which if its down not too much harm done, it's best that your first `POC` contains a server which if it's down no harm done.  do a full `docker` adoption on it and test it.  Once you feel comfort and you have formed procedures and templates for `docker` adoption you may continue gradually to servers with higher criticallity and with stricter `SLA`.  Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.




**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.


**Service discovery** We'll take `kubernetes` as the `scheduler and cluster manager`.  If you go through `kubernetes proxy` you can get access to the cluster by accessing any node of the kubernetes cluste on the ports which the `service` exposes.  However, you don't want to be dependant on the nodes themselfs.  So we can add another layer of abstraction in manner of an `external load balancer`.  Here we have 2 ways, we can query the `kubernetes api server` for the updated ports and ips of our service or we can define them as static in the service.  In anyway our client contacts a permanent `DNS` we can setup on our external load balancer and the load balancer will point to the service on the kubernetes cluster.   If you don't do that you are tightly coupled with the kubernetes cluster which you don't want to.  You want your system tests to run on a rather permanent set of locations on `QA` env (which can simply be `QA namespace`) and you can setup a different `DNS` for every env (namespace).

**Fast and Slow lane**  Your teams are already used to doing their tasks within the scopes of their roles, that is, development - providing tested artifacts to production while production team are in charge for their proper installation on production environments.  However, things are not always smooth, once you have a break fix, you have to consider how to perform a fix which should be both tested and delivered as fast as possible to production.  When dealing with artifacts and their deployment on production you should always have two lanes.  Fast and Slow.  SlowBy all means the slow lane is always used.  Always except for the time you have no other choice and you need to make a fix on production immediately.  You should aspire never getting to that place, but you have to know you have such an option.  Let's emphasize this again, as much as you should avoid this option you must have this option available, even theoretically so that if time comes it's available for you.  With `containerized deployments` the fast lane should be checkout the image in production, perform the fix, commit and apply it with a proper `tag` marking this was a break fix.  It is quick enough to enable you to discard the option of login into a container and doing the updates, we favour that, at least here you were working on a versioned controlled image, you can do edit the container directly, but its's much less recommended as we want to work on a clean image and commit it, while you do this in parallel you provide the change in the slow lane.  The slow lane includes updating the source code in development env, creating a new image which goes through the cycle of all `CI`, tests and providing this image back to production which should then install it as a proper image.

**Training** - You would need to train your developers AND QA AND support AND ANY other technical tier which has any access to servers in any env on docker.  Beside from learning syntaxly how to use docker you have to understand linux containers fundamentals, how docker is implements, how is memory shared in between your containers? disk? cpu? affinity? namespaces? how to access the processes outside of containers? how to read logs from your containers? how to permit developers access log files? Its not going to be a good idea that the troubleshooters won't know the environment on which their servers are running, good understanding that is.   There are plenty of items to check here and training, best thought of self training is the best option here.  The best way to achieve that would be to create some training receipes and lets the self training move on by itself.

<img src="https://docs.google.com/drawings/d/1QS6fZGaJuRIVfSG5bx5Sjq6E6C5xZHdyrluc3ycs7GM/pub?w=386&amp;h=252">

**Undrift your severs** So you have got everything in you favorite configuration management tool.  Every library is nicely and cozily upgraded and downgraded.  You would still have server drift.  First most of configuration managements do not handle deletion well, this already means you might have deletion drift.  If you ever ever do a manual fix (be it an internal library or patch) you are on the highway to server drift.  If all is managed by your configuration management tool then over time upgrades and downgrades which brings along dependendies and hence subdenencies will cause drift.  Once you start using container based images for deployments you are on the reverse highway to reverse that drift.  But as this is a game changer you should be aware of it.  And utilize it for the best, meaning, before patching anything or upgrading do it though the image, this will keep your servers from drifting.   

Note that as one of the main things we want to get out of docker or any other container is to prevent server drift, if you are already in a state where you are as immutable as you can in your server installations then you are in a good position.  If you are far from it first check how different are your servers one from another (which are supposed to be the same if they are running same app) if they are different then first check if by having them all the same your services are still behaving the same, this will have you much closer to being able to run docker with less issues at first.

**Troubleshooting container** containing tcpflow tcpdump and more tools.


**Appendix draft items not yet digested**
1. Networking - you have the internal docker port (process listening on port) the port docker exposes and the port kubernetes exposes and then the port your load balancer would expoese, ie F5.  if you have a docker container and you wish to check which ports it exposes and which ports are there internally you run `docker port containerid` or detect it via `kubernetes svc`



<img src="https://docs.google.com/drawings/d/1A3tDVCQf4LuPURHPgRrrBxEQhGgyDn69I_eE0mkQTao/pub?w=475&amp;h=336">

  
1. files logging Hi, what if i have kafka bridge that for resiliency writes files to disk.  Then it reads them and sends to kafka.  where should I be storing these files if i move that process into a pod?
                 
                 ​[6:07] 
                 if its emptyDir volume if my process moves to a new host machine it would loose the old history
                 
                 ​[6:08] 
                 if its to persistentVolume this would mean the path to the persistenVolume then as each container writes its own log of events to file which should be taken to kafka that would mean each container should have a volume with a convention agreed path like /path/kfakfa_log_files/some-process-id-like-349384/kakfa-logs/move-me-to-kafka-files
                 
                 ​[6:09] 
                 does this makes any sense? it sounds too complicated for me!! ALSO if its on NFS this would mean i'm again volnurable to network disconnctions which is the original reason why I wanted to write files to local disk.
1. There is no session affinity now in kubernetes.   nginx ---session affinity---> (app1 / app2 / app3 / app4 / ..).  We have many nginx with internal logic who access apps with sticky sessions (or any other arbitrary param we do now with nginx).  If we want to migrate these apps we must have sticky sessions.  I saw this which does not sound promising: https://github.com/kubernetes/kubernetes/issues/8731                   
1. where to place the Docker file take from presentation
2. logging.
3. Tagging labeling docker tag should just be the vesion name or latest registry is global and repository is for the project
4. monitoring the containers.
5. performance tuning.
6. using linux.
7. already using puppet? rpm? now you are going to have another scripting language docker md.
8. You could even increase security by having a container read only! nothing can write to it.  security audits can be faster all these apps are read only.
9. docker vs swarm vs kuber http://radar.oreilly.com/2015/10/swarm-v-fleet-v-kubernetes-v-mesos.html
10. What about rollout maybe consider the non rolled out services as all kubernetes service which will hide they are not a service its just an abstraction.
11. You can have another container the first one we don't need tcpdump and tcpflow and the other container will include these services which would mean you can listen to its ethernet without introducing security issues.


https://access.redhat.com/documentation/en/red-hat-enterprise-linux-atomic-host/version-7/getting-started-with-containers/

  [jinja-link]: http://jinja.pocoo.org/docs/dev/