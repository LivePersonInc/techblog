---
layout: post
title:  "Containers adoption pathway - Part X"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
published: true
---

# Docker adoption pathway - Part 2


**Training** - You would need to train your developers AND QA AND support AND ANY other technical tier which has any access to servers in any env on docker.  Beside from learning syntaxly how to use docker you have to understand linux containers fundamentals, how docker is implements, how is memory shared in between your containers? disk? cpu? affinity? namespaces? how to access the processes outside of containers? how to read logs from your containers? how to permit developers access log files? Its not going to be a good idea that the troubleshooters won't know the environment on which their servers are running, good understanding that is.   There are plenty of items to check here and training, best thought of self training is the best option here.  The best way to achieve that would be to create some training receipes and lets the self training move on by itself.

<img src="https://docs.google.com/drawings/d/1QS6fZGaJuRIVfSG5bx5Sjq6E6C5xZHdyrluc3ycs7GM/pub?w=386&amp;h=252">

**App and deployment types** You must be having many flavours of servers.  You must be having http style servers, but do you have also storm topologies? hadoop jobs? spark? spark streaming? `vert.x`? databases? which flavours? Do you already use `YARN`, `Mesos`? What about spontaneous command run like `tcpdump` do you also consider them as `apps`? (hint: yes).  This means you need to take into consideration a heterogenous production system and see how docker would fit it.  Also as you consider `docker` you probably already consider using `kubernetes` (hint: yes), This has to philarmon all together.

**Tagging Convention** You might have already been tagging your app releases.  By tags we don't mean plain software versions but also meaningful names to your builds.  Docker taken this step up to convention it already holds the notion of tags.  So now you can be officially a software release tagger, and if you wondered in the past whether that was a good enough practice here is an affirmation to that.  But, you ha
 the version control revision where this image was created from, tag can be extremely useful for that, so tag with the info you not to much but not too less, I would not recommend not including for example teh version control revision the image was created from, you really want a super easy way to get to the source code of that build.

# Docker adoption pathway - Part 3


**Fast and Slow lane**  Your teams are already used to doing their tasks, development - providing tested artifacts to production and production team for their proper installation on production environments.  Once you break the current methodology you have to consider the occasions where you need to perform a quick update on production.  Now that your app with OS conf is already bundled in an image, would you still be able to perform a quick fix when needed?  When dealing with artifacts and their deployment on production you should always have two lanes.  Fast and Slow.  By all means the slow lane is used always.  Always except for the time you have no other choice and you need to make the fix on production immediately.  As much as you should avoid this you must have this option available, even theoretically so that when time comes is available for you.  With containerized deployments the fast lane should be checkout the image in production, perform the fix, commit and apply it.  It is quick enough to enable you to discard the option of login into a container and doing the updates, you can do that, but you need to recall that this container would not nessesaraly live long enough to take on the change provided in the slow lane.  The slow lane includes updating the source code in development env, creating a new image which goes through the cycle of all tests and providing this image back to production.

**Undrift your severs** So you have got everything in you favorite configuration management tool.  Every library is nicely and cozily upgraded and downgraded.  You would still have server drift.  First most of configuration managements do not handle deletion well, this already means you might have deletion drift.  If you ever ever do a manual fix (be it an internal library or patch) you are on the highway to server drift.  If all is managed by your configuration management tool then over time upgrades and downgrades which brings along dependendies and hence subdenencies will cause drift.  Once you start using container based images for deployments you are on the reverse highway to reverse that drift.  But as this is a game changer you should be aware of it.  And utilize it for the best, meaning, before patching anything or upgrading do it though the image, this will keep your servers from drifting.   

Note that as one of the main things we want to get out of docker or any other container is to prevent server drift, if you are already in a state where you are as immutable as you can in your server installations then you are in a good position.  If you are far from it first check how different are your servers one from another (which are supposed to be the same if they are running same app) if they are different then first check if by having them all the same your services are still behaving the same, this will have you much closer to being able to run docker with less issues at first.


**configuration** for minimal configuration like one parameter pass it as argument, for a few more pass them as environment variable, always prefer to have convention for example if you use some cassandra host simply refer to DNS CASSANDRA in dev would be the right one in prod the right one.  For more complex configuration consider storing them in some kind of database or in volume files.  At first you see you have `yaml` files to define your `service` and `rc`.  But what if you need replication factor of one kind in one env and in another env a different `replication factor`?  You either end up with multiple duplicated `yaml` files or you `hack` something out.  The current recommended solution is to use `jinja2` as a templating tool so when you pass the `yaml` files for `kubernetes` to digest you have a few more `parameter` files *all under revision control ofcourse*.  You use the `diff` files as input to the `yaml` files so that you can run kubernetes on each env with its own specialized files. 

In addition where do you store those `yaml` configuration file? answer - your version control repository usually git repository.  You check them out on the machine you run `kubectl` on and run on them.

In addition what about your `app` configuration files? you might need also a different configuration file per `env`.  Ofcourse you should prefer `convetion over configuration` and if you still have a small set of configurations to pass to your app you can pass them through environment parameters or you can fetch them from a network storage like a `database`.  But what happens in the cases you still have local configuration files to configure for your app?  You should find some technique to `plant` those configuration files when your container is installed.  most probably via `volume` mounting.  You can mount the volume as a [gitRepo-volume](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/volumes.md#gitrepo), this means you store your configuration and manage their revisions in git repository.  Note that if you still have multiple configurations per multiple envs you need to utilize [jinja-templating][jinja-link]
 


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

`[jinja-link]: http://jinja.pocoo.org/docs/dev/`
.