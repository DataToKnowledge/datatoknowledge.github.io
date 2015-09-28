---
layout: post
title: Typesafe Config Late Resolving
description: "TypesafeConfig is a powerful library to deal with configurations I really like it but sometimes it makes me mad.
In this post I want to share how to deal in situation when you need late resolving for your configs."
modified: 2013-05-31
category: articles
tags: [scala typesafe akka docker]
imagefeature: cover6.jpg
comments: true
share: true
---

Typesafe Config Late Resolving
-----------------------

[Typesafe/Config](https://github.com/typesafehub/config) is a powerful library to deal with configurations I really like it but sometimes it makes me mad.

In this post I want to share how to deal in situation when you need late resolving for your configs.

### Example

Working on my project that uses [Akka](http://akka.io/) with [Docker](https://www.docker.com/)
and [Sbt-Native-Packager](http://www.scala-sbt.org/sbt-native-packager/) I need a way to:

1. detect the Docker container ip automatically when the builded images is ran
2. add the ip configuration to the "application.conf", and
2. to **resolve the config file**.

In the following there is the configuration file used.

{% highlight scala %}
include "common.conf"

akka {
  remote {
    netty.tcp {
      hostname = ${clustering.ip}
      port = ${clustering.port}
    }
  }

  cluster {
    roles = [feed-manager]
    seed-nodes = [
      "akka.tcp://"${clustering.clusterName}"@"${clustering.ip}":"${clustering.port}
    ]
  }
}

clustering {
  port = 5000
  clusterName = "ClusterSystem"
}

{% endhighlight %}

You can seed that I need to pass a `clustering.ip` configuration to the `seed-nodes` for the `cluster` configuration. Here the problem is that if I do something like

{% highlight scala %}
val config = ConfigFactory.parseString(
  s"""
    clustering.ip = ${ipAddress}
  """.stripMargin).
  withFallback(ConfigFactory.load("application.con"))
{% endhighlight %}

I get an error for the missing `clustering.ip` configuration.

### Solution

- Load a `baseConfig` allowing missing configurations and not resolving them.

{% highlight scala %}
import com.typesafe.config.{ ConfigParseOptions, ConfigResolveOptions, ConfigFactory }

val baseConfig = ConfigFactory.load(
  "application.conf",
  ConfigParseOptions.defaults.setAllowMissing(true),
  ConfigResolveOptions.defaults.setAllowUnresolved(true)
)
{% endhighlight %}

- loading the ip address

{% highlight scala %}
val ipAddress = HostIp.load().getOrElse("127.0.0.1")
print(s"detected ip $ipAddress")
{% endhighlight %}

- add the configuration and call the resolve method

{% highlight scala %}
val config = ConfigFactory.parseString(
  s"""
    clustering.ip = ${ipAddress}
  """.stripMargin).
  withFallback(baseConfig).resolve
{% endhighlight %}

Here it is! Now the ip address is correctly resolved and the `cluster` configuration is correctly evaluated. 

{% highlight scala %}
  cluster {
    roles = [feed-manager]
    seed-nodes = [
      "akka.tcp://"${clustering.clusterName}"@"${clustering.ip}":"${clustering.port}
    ]
  }
{% endhighlight %}

### Conclusion
To get a complete example of the solution you can check the [wheretolive-feed project](http://github.com/DataToKnowledge/wheretolive-feed) on [Github](http://github.com).

Another solution can be passing the needed parameters via command line.

{% highlight bash %}
-Dapp.initialContacts.0 = "akka.tcp://ClusterSystem@127.0.0.1:5000/user/receptionist"
{% endhighlight %}

