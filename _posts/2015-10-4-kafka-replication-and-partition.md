---
layout: post
title: "Kafka topic: replication factor and partitioning"
description: "This post explains how to setup a kafka topic with replication and partitioning"
modified: 2015-10-04
author: DTK
category: Guide
tags: [guide, kafka, topic]
comments: true
share: true
---

In this post I will briefly explain how to manage kafka topic replication factor and partitioning.

##Create new topic specifying replication factor and partitions
Suppose we need to create a kafka topic with 3 replica and 2 partitions named `foo`

###Using kafka-topic script
The easiest way to create a topic is by using `bin/kafka-topic.sh` script, we can do this with:

```bash
$ bin/kafka-topic.sh --create --zookeeper <zk_host>:<port> --replication-factor 3 --partitions 2 --topic foo
```

###Using org.apache.kafka library with Scala
In Kafka 0.8.1+ you can programmatically create a new topic via `AdminCommand`.
First of all you need to import `org.apache.kafka` in your `sbt` project:

```Scala
name := "test-kafka-project"
...
libraryDependencies ++= Seq(
  "com.101tec" % "zkclient" % "0.4",
  "org.apache.kafka" %% "kafka" % "0.8.2.1",
  ...
)
...
```
then you can use the `createTopic` method of `AdminCommand`:

```Scala
import kafka.admin.AdminUtils
import kafka.utils.ZKStringSerializer
import org.I0Itec.zkclient.ZkClient

// Create a ZooKeeper client
val sessionTimeoutMs = 10000
val connectionTimeoutMs = 10000
val zkClient = new ZkClient("zoo-1:2181", sessionTimeoutMs, connectionTimeoutMs,
    ZKStringSerializer)

// Create a topic named "myTopic" with 8 partitions and a replication factor of 3
val topicName = "foo"
val numPartitions = 2
val replicationFactor = 3
val topicConfig = new Properties
AdminUtils.createTopic(zkClient, topicName, numPartitions, replicationFactor, topicConfig)
```

##Increase replication factor and/or partitions for an existing topic
Suppose we havea a kafka topic named `boo` with 1 replica and 1 partitions.

###Increase replication factor
Increasing the replication factor of an existing partition is easy. Just specify the extra replicas in the custom reassignment json file and use it with the `--execute` option to increase the replication factor of the specified partitions.

The first step is to hand craft the custom reassignment plan in a json file:

```bash
$ echo {"version":1, "partitions":[{"topic":"boo","partition":0,"replicas":[1,2,3]}]} > increase-replication-factor.json
```

In this example we have 3 kafka broker with id 1, 2 and 3.
Once we have our reassignment json file we can increase replication factor with:

```bash
$ bin/kafka-reassign-partitions.sh --zookeeper <zk_host>:<port> --reassignment-json-file increase-replication-factor.json --execute

Current partition replica assignment

{"version":1, "partitions":[{"topic":"boo","partition":0,"replicas":[2]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions
{"version":1, "partitions":[{"topic":"boo","partition":0,"replicas":[1,2,3]}]}
```

The `--verify` option can be used with the tool to check the status of the partition reassignment.

```bash
$ bin/kafka-reassign-partitions.sh --zookeeper <zk_host>:<port> --reassignment-json-file increase-replication-factor.json --verify

Status of partition reassignment:
Reassignment of partition [boo,0] completed successfully
```

###Add partitions
To add partitions you can do

```bash
> bin/kafka-topics.sh --zookeeper <zk_host>:<port> --alter --topic boo --partitions 2
```
**NB:**
Be aware that one use case for partitions is to semantically partition data, and adding partitions doesn't change the partitioning of existing data so this may disturb consumers if they rely on that partition.

Kafka does not currently support reducing the number of partitions for a topic or changing the replication factor.
