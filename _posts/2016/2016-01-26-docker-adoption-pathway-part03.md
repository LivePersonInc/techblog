---
layout: post
title:  "Docker adoption pathway - Part 3"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: true
---
# Docker adoption pathway - Part 3

## Introduction

In [Docker adoption pathway Part 2]() we discussed the context of `cpu` monitoring in `containers`. We examined choices and different paths for monitoring `containers cpu`.  In addition, we covered the way `docker` computes the `cpu` consumption of a `container`. We summarized by looking at how the `API` docker exposes the way in which we can collect the container's `cpu` metrics. We went through that path in order to properly monitor our container as part of our standard `data-center` monitoring process. In this post, we are going to continue the `monitoring` track, however, this time we will look at how `JMX` is used for monitoring in containerized environments. `JMX` is heavily used as a successful Java applications management and monitoring tool and has been over the years since its inception. This means we have to consider its behavior in our new containerized world. By utilizing `JMX` we can glance inside our running Java application to collect and analyze behavior. Then we initiate a `JMX` connection to a Java application. This may sound trivial, but it requires more than a few steps, problems, and possibilities for mitigation to reach a successful outcome. We’ll go through the process. 

**JMX and Containers**

**What could possibly be different with `containers`?** You ask a valid question in terms of this and many other transitions of apps to be containerized. Why would exposing and using plain `JMX` service from a `container` be any different from exposing and using it on a standard `VM` process? 

These questions are certainly valid and, as we shall see, they touch a few sensitive issues with containers. Namely, how do we configure processes in containers? Do we configure them at all, or do we ship containers together with their configurations as immutable packages? What is the relationship between `orchestrator services` and management services such as `JMX`? How should I allow GUI apps outside the `containers` cluster to access `JMX` services that inherently reside inside the `cluster`? From a deployment perspective, JMX translates to an exposed port. `Containers` have their own `port` exposure dynamics. `Containers’ orchestrators` have services. The typical deployment architecture of a `container` is an application as a `container` sitting behind either `web apps` or plain `orchestrator services`. Services such as Kubernetes container management system provides round robin routing for the apps backend, which is not suitable when there is a need to access a specific container `JMX`.  

In this post, we are going to run Java application, Apache Cassandra, and try to connect to its `JMX` while it's being run as a container. While most of the changes we are going to perform in order to be able to connect to our Cassandra container do not appear to be directly related to `docker`, most of them affect the way we package and run our container so they end up in a close relationship with our containers packaging and deployment.

Below is an example:

**Let's connect to a container's JMX**

To begin with, we are going to launch a Cassandra instance as a `container` (of course).  Fortunately, using `docker`, this can be done using a single line of command: 

```bash
docker run --name containerized-cassandra -d cassandra 
91523e6a1e34f52e89993ae75821633a92b2528c5e0f551983a9518f7044d286
```

This `Cassandra` instance serves as our Java application example, which is now up and running as a `docker container`.   

Let's connect to its `JMX` interface. `Cassandra` exposes JVX, so what's the port, you ask? According to its documentation, it is on: [7199 - Cassandra JMX monitoring port.](https://docs.datastax.com/en/cassandra/2.0/cassandra/security/secureFireWall_r.html)
 
We are going to connect to the JMX like a pro, which means, not with UI but with a command line! For this, someone already prepared a nice utility - [jmxterm jmx command line client](http://wiki.cyclopsgroup.org/jmxterm/) - we are going to ask it to connect to the Cassandra JMX using this command:

```bash
$ java -jar ~/Downloads/jmxterm-1.0-alpha-4-uber.jar  --url localhost:7199
```

Alas! The results:

```bash
java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.ServiceUnavailableException [Root exception is java.rmi.ConnectException: Connection refused to host: localhost; nested exception is: 
	java.net.ConnectException: Connection refused]
```

With a `Connection Refused` error, we understand there is no service on the other side listening on that host/port (or there is service, but we have no way to access it). This means the service does not expose its port for us. For `JMX` we can utilize `telnet` and check to see if we have any connection to `Cassandra` via the `JMX port` which is `7199`:

```bash
telnet localhost 7199
:~/tmp/java-docker$ telnet localhost 7199
Trying 127.0.0.1...
telnet: Unable to connect to remote host: Connection refused
```

No connection. If we look at [`cassanra's Dockerfile`](https://github.com/docker-library/cassandra/blob/90ba62c6d8859abc5f38a6d47c9da0661be04171/3.3/Dockerfile#L43) we see that it does `expose` port `7199`:

```Dockerfile
# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
EXPOSE 7000 7001 7199 9042 9160
```
 
However, it is not going to be exposed to an `unlinked container` or to our `visualvm/jconsole` from outside the container environment. For that purpose, when we `run` the `Cassandra container` we are going to use the `-p` option in order to `expose` that port also on the `node` itself. Our `docker run` command would be as follows:

```bash
docker run --name containerized-cassandra -p 7199:7199 -d cassandra
```

When we run `docker ps` we can verify that the port was exposed:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                       NAMES
11b499d74c4e        cassandra           "/docker-entrypoint.s"   25 seconds ago      Up 24 seconds       7000-7001/tcp, 9042/tcp, 0.0.0.0:7199->7199/tcp, 9160/tcp   containerized-cassandra
```

You see that the only port exposed is the JMX port: `0.0.0.0:7199->7199/tcp`

If we now check to see if that port is exposed, we learn that it is:

```bash
$ telnet localhost 7199
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```

Now let’s try again with `jmxterm`:

```bash
java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: error during JRMP connection establishment; nested exception is: 
	java.net.SocketException: Connection reset]
```

We receive a `connection reset`. Before, we received a `connection refused` because no port was exposed, and now we receive a `connection reset`.  Let's log in to the image and try to understand why it happened:

```bash
$ docker exec -it containerized-cassandra bash
```

Let's find the startup script to Cassnadra and see where it configures `jmx`:

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

We see that Casssandra has a `cassandra-env.sh`. It's used to configure the already started Cassandra instance. Let’s look at it and see how it configures the `jmx`:

```bash
root@104f281f5f09:/# more /etc/cassandra/cassandra-env.sh
```

…and we find this interesting response:

```bash
if [ "$LOCAL_JMX" = "yes" ]; then
  JVM_OPTS="$JVM_OPTS -Dcassandra.jmx.local.port=$JMX_PORT -XX:+DisableExplicitGC"
else
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.port=$JMX_PORT"
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.rmi.port=$JMX_PORT"
  JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.ssl=false"
```

…so, we learn if we do `ps -ef | grep cassandra` we find that we have `-Dcassandra.jmx.local.port=7199` which means we are using the `local.port` and not setting the `com.sun.management.jmxremote.rmi.port`. This means that `LOCAL_JMX` is `yes`.  Therefore, we are going to change `LOCAL_JMX` to `no` by means of passing an environment variable to our Cassandra docker image.

```bash
$ docker run --rm --name containerized-cassandra -p 7199:7199 -e LOCAL_JMX='no' cassandra
```

Now we get `Error: Password file not found: /etc/cassandra/jmxremote.password`. Let's see if the folder `/etc/cassandra` already exists in the container because, if it's not, we are free to mount it:
  
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

There are multiple paths to take here:

1. Contribute to the original Cassandra `image` with an update specifying where the `jmxremote.password` exists.
2. Mount the `/etc/cassandra` externally with a `volume` to allow you to edit these configurations externally.
3. Extend the `cassandra` image and add to it your own `jmxremote.password` file.
4. Connect the JMX passwords to your `LDAP` service.

While the `LDAP` service is our favorite option, even though this is a repeating theme, local changes are moving to the network. For this example we are going to create our own image and add to it the `jmxremote.password` file.

We create a new `jmxremote.password`:

```bash
controlRole somepass
```

Our `Dockerfile` would appear as follows:

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

So, we restrict the file access, `Dockerfile`:

```bash
FROM cassandra
COPY jmxremote.password /etc/cassandra/
RUN chmod -R 600 /etc/cassandra/jmxremote.password
```

We again build the image:

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

and now login to JMX with the `user` we defined:

```bash
$ java -jar ~/Downloads/jmxterm-1.0-alpha-4-uber.jar --user controlRole --url localhost:7199
Authentication password: ********
Welcome to JMX terminal. Type "help" for available commands.
$>
```

SUCCESS! We have connected to our `containerized Cassandra JMX`.

