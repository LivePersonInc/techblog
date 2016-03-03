---
layout: post
title:  "Docker adoption pathway - Part 3"
date:   2016-01-26 22:18:00
categories: software-development,architecture,design
comments: true
author: Tomer Ben-David
published: false
---
# Docker adoption pathway - Part 3

**configuration** 

A `container` as it's name promises is a self container application to get deployed and run.  This raises a question, if its self contained how would configuration which by it's nature is an external dynamic input to the container be encapsulated and shipped into the container to change its behaviour?  As always there is no one magical bullet to achieve that.  In this post we will scan our options discussing when should we use each option.

First off let's discuss which app types configuration we have, in general we can divide configuration types to:

1. 

for minimal configuration like one parameter pass it as argument, for a few more pass them as environment variable, always prefer to have convention for example if you use some cassandra host simply refer to DNS CASSANDRA in dev would be the right one in prod the right one.  For more complex configuration consider storing them in some kind of database or in volume files.  At first you see you have `yaml` files to define your `service` and `rc`.  But what if you need replication factor of one kind in one env and in another env a different `replication factor`?  You either end up with multiple duplicated `yaml` files or you `hack` something out.  The current recommended solution is to use `jinja2` as a templating tool so when you pass the `yaml` files for `kubernetes` to digest you have a few more `parameter` files *all under revision control ofcourse*.  You use the `diff` files as input to the `yaml` files so that you can run kubernetes on each env with its own specialized files. 

In addition where do you store those `yaml` configuration file? answer - your version control repository usually git repository.  You check them out on the machine you run `kubectl` on and run on them.

In addition what about your `app` configuration files? you might need also a different configuration file per `env`.  Ofcourse you should prefer `convetion over configuration` and if you still have a small set of configurations to pass to your app you can pass them through environment parameters or you can fetch them from a network storage like a `database`.  But what happens in the cases you still have local configuration files to configure for your app?  You should find some technique to `plant` those configuration files when your container is installed.  most probably via `volume` mounting.  You can mount the volume as a [gitRepo-volume](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/volumes.md#gitrepo), this means you store your configuration and manage their revisions in git repository.  Note that if you still have multiple configurations per multiple envs you need to utilize [jinja-templating][jinja-link]
 
resources:
https://dantehranian.wordpress.com/2015/03/25/how-should-i-get-application-configuration-into-my-docker-containers/