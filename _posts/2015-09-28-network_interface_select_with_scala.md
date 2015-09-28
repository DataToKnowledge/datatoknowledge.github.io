---
layout: post
title: How to select a network interface from scala
description: This post explain how to access to network interfaces in linux from scala
modified: 2015-09-28T00:00:00.000Z
category: articles
tags:
  - scala typesafe akka docker network interface
imagefeature: cover6.jpg
comments: true
share: true
---

During my last project [wheretolive-feed](https://github.com/DataToKnowledge/wheretolive-feed) I need to select the network interface to use [Akka Cluster](http://doc.akka.io/docs/akka/snapshot/scala/cluster-usage.html) with [Docker](https://www.docker.com/).<br>I tried with the solution showed in [akka-docker-cluster-example](https://github.com/mhamrah/akka-docker-cluster-example) which injects a bash command in the docker entrypoint, but it does not works with the last Docker image for java.

```
dockerEntrypoint in Docker := Seq("sh", "-c", "CLUSTER_IP=`/sbin/ifconfig eth0 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1 }'` bin/clustering $*")
```

# Proposed solution
Thus, my solution is based on:
1. enumerating the network interfaces bind to the host
2. to inject the selected network interface by name

```scala
import scala.collection.JavaConversions._
import java.net.NetworkInterface

/**
 * Created by fabiofumarola on 16/09/15.
 */
object HostIp {

  def findAll(): Map[String, String] = {
    val interfaces = NetworkInterface.getNetworkInterfaces
    interfaces.flatMap { inet =>
      inet.getInetAddresses.
        find(_.isSiteLocalAddress).
        map(e => inet.getDisplayName -> e.getHostAddress)
    } toMap
  }

  def load(name: String): Option[String] = {
    val interfaces = NetworkInterface.getNetworkInterfaces
    val interface = interfaces.find(_.getName == name.toString)

    interface.flatMap { inet =>
      inet.getInetAddresses.find(_.isSiteLocalAddress) map (_.getHostAddress)
    }
  }
}
```

- The method `findAll` enumerate all the available network interfaces
- The method load can be used in the main code of your application to detect the right IP at runtime.

Finally you can combine this solution with what is described in my previous post [typesafe_config_late_resolving](typesafe_config_late_resolving).
