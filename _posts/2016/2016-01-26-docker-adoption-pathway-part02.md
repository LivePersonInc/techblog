---
layout: post
title:  "Docker adoption pathway - Part 2"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: true
---
In [part 1](http://livepersoninc.github.io/techblog/docker-adoption-pathway-part01.html) we went through the **motivation** for `containers` adoption, we checked our **readiness** for `immutable deployments` and went through some of the **effect on current deployment** tools that are already in place.  To recap, converting new services in new environments for containers is very different from converting existing or new services with existing environments, deployment methodology and troubleshooting procedures; the latter including much more effort.  Therefore, in this part we are going to see how troubleshooting may change, examine which apps are good candidates to get `containerized` first, and, lastly, we are going to examine how `service discovery` changes. 

**Troubleshooting** 

**Remote troubleshooting** 

Today, much of troubleshooting is done **remotely**, meaning you usually do not access locally the hosts. tools such as **`graphite`, `ELK`, `newrelic`**, we also have custom home made tools which handle sending information remotely and on the other side viewing them.   We are getting used that instead of accessing directly servers we find information about them remotely.   This `remote troubleshoot mostly` methodology stays the same.

**When you can't remote troubleshoot** 

There are cases however, when you find that you need to access your servers and run various troubleshooting commands such as **`uptime`, `ps`, `lsof`, `tcpdump`, `dmesg`, `sar`**.  All this in order to check nodes and apps health and find problems root cause, standard troubleshooting.  In addition, many times you expose custom web management pages on your `app` instance which exposes your app's internal state and data or metrics on your instances which possibly allows you to run commands directly on your app instance when needed.  In `jvm` based `apps` in addition to these management web pages it is very common to have **`jmx`** exposed to check the status of the app and manage it, that is what `jmx` was built for after all.  

**When you have your app `containerized` things change**.  First and foremost **dealing with machines**, you deal less with machines and deal more with the **`cluster orchestrator`**, be it `kubernetes`, `fleet`, `docker compose` or other orchestration tools.  With regards to having the web pages or `jmx` exposed orchestrator services take care of exposing ports which are used to access your server as a logical business unit, but, what about accessing ports on your specific app node? the **`services`** usually perform load balancing, example `round robin` **load balancing**, this does not help when you need to access a specific app and not only one of the apps.  If the gateway which runs `jconsole` is outside the cluster you would need to expose the ports so that that gateway has access to it.  If you choose, with another layer of abstraction we can create a tool in our cluster where we would send it commands or request to see apps management web pages and it would return us the page or jmx result of a specific container.  

We are going to check out 3 examples of troubleshooting in the new world:

1. Performance - **`cpu metrics collection`**
1. Applicative - **`running jmx`**
1. utility commands - **`running tcpdump`**

**3 examples** We are going to check out 3 different aspects of troubleshooting: applicative - `running jmx`, performance: `cpu metrics collection`, `utility commands` - tcpdump.

**Example 1 Monitoring CPU** 

Create dockerfile which utilizes cpu `Dockerfile`:

```bash
FROM ubuntu
COPY looper.sh .
RUN chmod +x ./looper.sh
CMD ["./looper.sh"]
```

looper is a simple single core killer `looper.sh`
                                      
```bash
#!/bin/bash
while [ true ] ;do echo `date`;done
```


build the docker image:

```bash
$ docker build -t "looper" .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM ubuntu
 ---> 0dd3b242dfe5
Step 2 : CMD while [ true ] ;do echo `date`;done
 ---> Running in 8cd8baca53f9
 ---> dbb75598f34a
Removing intermediate container 8cd8baca53f9
```

run it and allow it to run only on a **single core**:

```bash
$ docker run --name="single-cpu-killer" --rm --cpuset-cpus="0" looper
```

As we know our docker container is a process let's search it in list of processes:

```bash
$ ps -ef | grep looper.sh
root     12454  2335 16 17:39 ?        00:20:25 /bin/bash ./looper.sh
```

let's use the `ps` command to find its cpu utilization.

```bash
$ ps -p 12454 -o %cpu 
%CPU
17.0
```

Wait! we have 4 cores and our process fully utilizes one core at least that's what we asssume so shouldn't utilization be closer to 25%? lets check the manual on `ps` the man page says: `CPU usage is currently expressed as the	percentage of time spent running during the entire lifetime of a	process. This is not ideal, and	it does	not conform	to the standards that ps otherwise conforms to.	CPU usage is unlikely to add up to exactly 100%.`.  this can explain it, but we can verify this, there are more methods to check for cpu utilization.  As we mentioned earlier we always prefer to use the standard tools, lets use docker stanard tool for cpu utilization measurment:

We are going to first utilize docker commands for inspecting the cpu:

```bash
docker stats single-cpu-killer
```

which results with:

```bash
CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O
single-cpu-killer   97.90%              413.7 kB / 16.52 GB   0.00%               3.882 kB / 648 B    0 B / 0 B
```

so though we have 4 cores the docker stats for that docker process shown close to `100%` utilization which makes sense because it treats that process as the only process.

if we want to see how actually the `docker stat` calculates the `cpu usage percentage` we should have a look at `docker client` sources at `stats.go`

[docker client stats.go](https://github.com/docker/docker/blob/master/api/client/stats.go)

we can see first a call to calculate cpu percentage:

```go
previousCPU = v.PreCPUStats.CPUUsage.TotalUsage
previousSystem = v.PreCPUStats.SystemUsage
cpuPercent = calculateCPUPercent(previousCPU, previousSystem, v)
```

and then `func calculateCPUPercent(previousCPU, previousSystem uint64, v *types.StatsJSON) float64 {` with:

```go
cpuDelta = float64(v.CPUStats.CPUUsage.TotalUsage) - float64(previousCPU)
systemDelta = float64(v.CPUStats.SystemUsage) - float64(previousSystem)
```

Note that the `systemCPU` here is not the system cpu as used by the container, but the total machine system cpu delta.  While the cpuDelta is the `container` cpu delta for the container.  This means we can divie the `containerCPU` which is `totalUsage` diff by the `systemCPU` which is the total diff and this is exactly what is done at: **(cpuDelta / systemDelta)** see the following `calculateCPUPercent` which takes as argument `prevCPU` as used by container `previousSystem` total system cpu and current cpu usages at `v`:

```go
func calculateCPUPercent(previousCPU, previousSystem uint64, v *types.StatsJSON) float64 {
	var (
		cpuPercent = 0.0
		// calculate the change for the cpu usage of the container in between readings
		cpuDelta = float64(v.CPUStats.CPUUsage.TotalUsage) - float64(previousCPU)
		// calculate the change for the entire system between readings
		systemDelta = float64(v.CPUStats.SystemUsage) - float64(previousSystem)
	)

	if systemDelta > 0.0 && cpuDelta > 0.0 {
		cpuPercent = (cpuDelta / systemDelta) * float64(len(v.CPUStats.CPUUsage.PercpuUsage)) * 100.0
	}
	return cpuPercent
}
```

As we might have multiple `cores` we need to multiply by PercpuUsage and then convert to percentage by multiplying by 100.0.

Underneath how is this cpu data being taken?  docker uses `cgroups` (`control groups`) to control process resources, it controls `CPU, memory, diskio, network, etc` for process groups.

for example when we ask docker to provide only `50%` of the cpu by using `--cpu-quota="50000"` we actually update the cpu_quota in cgroups, see:

```bash
$ docker run --name="single-cpu-killer" --rm --cpuset-cpus="0" --cpu-quota="50000" looper
```

and if we use the standard `docker stat` we see its consuming `50%` of the single core:

```bash
CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O
c376f04ad25a        48.69%              3.981 MB / 16.52 GB   0.02%               10.21 kB / 648 B    10.84 MB / 0 B
```

and the way docker has achieved that is by updating the `cpu.cfs_quota_us` for that process

```bash
cat /sys/fs/cgroup/cpu/docker/c376f04ad25aaa54eec4f2afe63579127bd836a392ef9b4d2b153d47bd5adc62/cpu.cfs_quota_us 
50000
```

Let's locate our container in `cgroups`

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
8df2eab576ae        looper              "./looper.sh"            About an hour ago   Up About an hour                                                  single-cpu-killer
```

the cpu accounting information would reside at: `/sys/fs/cgroup/cpuacct`
 
and for our container:

```bash
cat /sys/fs/cgroup/cpuacct/docker/8df2eab576ae064b410dc24fbd19d56c3621bedc7b615646b7ac62ab63a1aa67/cpuacct.usage
6285931657516
```

this is absolute CPU usage of the container in nanoseconds.  If it equals the number of nanoseconds (for a single core) in 1 second this means its in 100% cpu usage.

so lets print this number of our container in time diff of 1 second:

```bash
$ cat /sys/fs/cgroup/cpuacct/docker/8df2eab576ae064b410dc24fbd19d56c3621bedc7b615646b7ac62ab63a1aa67/cpuacct.usage;sleep 1;cat /sys/fs/cgroup/cpuacct/docker/8df2eab576ae064b410dc24fbd19d56c3621bedc7b615646b7ac62ab63a1aa67/cpuacct.usage

6379039984585
6379982913877
```

take the difference between those two numbers and divide by number of nanoseconds in one second (1 billion) and you get indeed 94% cpu utilization.

So we know how to get the cpu percentage with `docker stats` command, but that's still not that useful for monitoring systems, just as we don't want to parse top we would rather get something more structured such as a json.  Thankfully docker provides remote http api which we can utilize for that purpose we use:

```bash
$ echo -ne "GET /containers/single-cpu-killer/stats HTTP/1.1\r\n\r\n" | sudo nc -q -1 -U /var/run/docker.sock
```

and we get back all the cpu information with json:

```json
"cpu_stats":{  
   "cpu_usage":{  
      "total_usage":222305427989,
      "percpu_usage":[  
         222305427989,
         0,
         0,
         0
      ],
      "usage_in_kernelmode":159780000000,
      "usage_in_usermode":62740000000
   },
   "system_cpu_usage":6215130000000,
   "throttling_data":{  
      "periods":4665,
      "throttled_periods":4663,
      "throttled_time":230511526966
   }
},
"memory_stats":
```

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

