---
layout: post
title:  "Docker adoption pathway - Part 2"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: true
---
**Introduction**

In [part 1](http://livepersoninc.github.io/techblog/docker-adoption-pathway-part01.html) we went through the motivation for `containers` adoption; we checked our readiness for `immutable deployments` and also examined the effect on current deployment tools that are already in place.  To recap, converting new services in new environments for containers is very different from converting existing or new services with existing environments, deployment methodology and troubleshooting procedures; with the latter including much more effort.  In this part, we are going to see how troubleshooting changes when adopting `containerized` environments.

**Troubleshooting in containerized environments** 

**Troubleshoot remotely**

Today, much of troubleshooting is done remotely, meaning, you usually do not access the hosts locally.  tools such as **`graphite`, `ELK`, `newrelic`** are very common remote troubleshooting tools.  In addition, companies develop custom inhouse tools which handle sending information from local machines to remote server, having users view the results with various clients.   We are already got used to the fact that instead of accessing directly servers we find information about them remotely.   This `remote troubleshoot mostly` methodology stays the same.

**When you can't remote troubleshoot** 

There are cases however when you find that you need to access your servers and run various troubleshooting commands such as `uptime`, `ps`, `lsof`, `tcpdump`, `dmesg`, `sar` locally.  All this in order to check nodes and apps health and find problems root cause - standard troubleshooting.  In addition, many times you expose custom web management pages on your `app` instance which exposes your app's internal state and data or metrics which possibly allows you to run commands directly on your app instance when needed.  In `jvm` based `apps` in addition to these management web pages it is very common to have `jmx` exposed to check the status of the app and manage it - that is what `jmx` was built for after all.  

**When you have your app `containerized` things change**.  First and foremost dealing with machines, you deal less with machines and deal more with the `cluster orchestrator`.  With regards to having the web pages or `jmx` exposed orchestrator services take care of exposing ports which are used to access your server as a logical business unit, but, what about accessing ports on your specific app node? the `services` usually perform load balancing, example `round robin` load balancing, this does not help when you need to access a specific app and not only one of the apps.  If the gateway which runs `jconsole` is outside the cluster you would need to expose the ports so that that gateway has access to it.  If you choose, with another layer of abstraction you can create a tool in the cluster where you would send it commands or request to see apps management web pages and it would return the page or jmx result of a specific container.  

In this post we are going to focus on `cpu metrics collection` scenario.

**Monitoring CPU**

In this scenario we are going to check for ways to collect cpu metrics and what we want to take into account when working in containerized environment.

Questions to ask:

1. What does the `machine cpu` metric tells us now that we have containers starting and stopping dynamically? is high cpu bad or does it simply  means the orchestration is managing to make good usage of our metal?
1. How do you collect `container cpu`?
1. With the multiple ways we find for collecting `container cpu` is there a difference in the actual data collected? can we get different results?
1. Is there a parsable friendly format for `cpu metric` collection?

We are going to work out these questions by examples.  First we create `Dockerfile` which can fully stress a single cpu core:

```bash
FROM ubuntu
COPY looper.sh .
RUN chmod +x ./looper.sh
CMD ["./looper.sh"]
```

Looper is a simple single core killer `looper.sh`:
                                      
```bash
#!/bin/bash
while [ true ] ;do echo `date`;done
```


Build the docker image:

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

Run it and allow it to run only on a single core:

```bash
$ docker run --name="single-cpu-killer" --rm --cpuset-cpus="0" looper
```

As we know our docker container is a process let's search it in list of processes:

```bash
$ ps -ef | grep looper.sh
root     12454  2335 16 17:39 ?        00:20:25 /bin/bash ./looper.sh
```

Let's use the `ps` command to find its cpu utilization.

```bash
$ ps -p 12454 -o %cpu 
%CPU
17.0
```

Wait! we have 4 cores and our process fully utilizes one core at least that's what we assume.  Shouldn't utilization be closer to 25%? lets check the manual on `ps` the man page says: `CPU usage is currently expressed as the percentage of time spent running during the entire lifetime of a process. This is not ideal, and it does not conform to the standards that ps otherwise conforms to.  CPU usage is unlikely to add up to exactly 100%.`.  this can explain it, but we can verify this, there are more methods to check for cpu utilization.  As we mentioned earlier we always prefer to use the standard tools, lets use docker standard tool for cpu utilization measurement.

We are going to first utilize docker commands for inspecting the cpu:

```bash
docker stats single-cpu-killer
```

which results with:

```bash
CONTAINER           CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O
single-cpu-killer   97.90%              413.7 kB / 16.52 GB   0.00%               3.882 kB / 648 B    0 B / 0 B
```

So though we have 4 cores the docker stats for that docker process shown close to `100%` utilization which makes sense because it treats that process as the only process.

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

Note that the `systemCPU` here is not the system cpu as used by the container, but the total machine system cpu delta.  While the cpuDelta is the `container` cpu delta for the container.  This means we can divide the `containerCPU` which is `totalUsage` diff by the `systemCPU` which is the total diff and this is exactly what is done at: (cpuDelta / systemDelta) see the following `calculateCPUPercent` which takes as argument `prevCPU` as used by container `previousSystem` total system cpu and current cpu usages at `v`:

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

We can see that we can get both cpu and memory (and additional) metrics for our docker container by connecting to the docker remote api.  The results are parsable which is great as we can pinpoint the information we need for presentation in monitoring components.  And this is the method we adopt for monitoring containers cpu.  We haven't fully discussed what does machine `cpu` means (if it means..) and what should we do in cases we have to run local commands on hosts, we are going to leave that for future posts.