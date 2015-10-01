---
layout: post
title: Setup Weave 1.1.0 on Multiple VMs
description: "This post explains how to setup a private network on multiple cloud providers using docker and weave"
modified: 2015-10-01
category: guide
tags: [docker, weave, network, how to]
imagefeature: cover6.jpg
comments: true
share: true
---

[weave.works](http://weave.works/) solves the problem of creating networks of Docker containers without opening several ports to allow them to communicate. This post is the getting started of a tutorial in which we will show how:

1. to create a cluster of docker that can communicate across VMs without opening any other port except the 6783 tcp/udp, 
2. to give access to your services using a reverse proxy and weave dns
3. to use an NGINX instance as single point of access to your services

## Rationale

We come to [Weave]((http://weave.works/)) when we got [Azure MSDN](https://azure.microsoft.com/it-it/pricing/member-offers/msdn-benefits-details/) subscription and we had to manage several subscriptions. At first, we try to map each service to each port but this was a problem when you have service such as Spark or Elasticsearch that decide communication ports at runtime. 

##How to install Weave on Ubuntu

This tutorial is based on [Ubuntu Weave Tutorial](http://weave.works/guides/weave-docker-ubuntu-simple.html), but is rewritten for [weave 1.1.0](https://github.com/weaveworks/weave/releases).

**Prerequisite:** Linux or Mac, VirtualBox, Vagrant and Git

Let us clone this repository.

```bash
git clone http://github.com/weaveworks/guides
cd guides/ubuntu-simple
vagrant up
```
These steps start two VMs `weave-gs-01` and `weave-gs-02`. Once the VMs are ready we can check their status and log into

```
vagrant status

weave-gs-01               172.17.8.101
weave-gs-02               172.17.8.102
```

### Installing Weave

```
vagrant ssh weave-gs-01
sudo -s
curl -L git.io/weave -o /usr/local/bin/weave
chmod a+x /usr/local/bin/weave
```

```
vagrant ssh weave-gs-02
sudo -s
curl -L git.io/weave -o /usr/local/bin/weave
chmod a+x /usr/local/bin/weave
```

### Starting Weave
We are running all the command as `sudo` user please don't do that in production, add weave to sudoers such as docker

Now we can create a peer connection between the two hosts.

On host weave-gs-01

```
weave launch 172.17.8.102
```
On host weave-gs-02

```
weave launch 172.17.8.101
```
`weave launch` launch the [router](), [weaveDNS]() and the proxy service, that we will deeply describe in next posts.

```
weave status

weave status peers

e2:0b:99:36:a8:5d(weave-gs-01)
   <- 172.17.8.102:35664    2e:5d:93:a5:25:09(weave-gs-02)   established
2e:5d:93:a5:25:09(weave-gs-02)
   -> 172.17.8.101:6783     e2:0b:99:36:a8:5d(weave-gs-01)   established
```

`weave status peers` shows the correct list of the nodes connected by weave. 

### Simple Docker test with weave

In this example we will run a docker in `weave-gs-01` and ping it by name from a docker ran in `weave-gs-01`.

In order to make the ran container exposed into docker we need to perform the following command

```
eval $(weave env)
```
otherwise the container will be not exposed in the weave net.

```bash
#host weave-gs-1
docker run --name=pingme -dit ubuntu

#host weave-gs-2
docker run -it ubuntu
```

With the first command we started a docker container with name pingme. This name is also used as `/etc/hosts` entry and a record is stored by the proxy in the dns. Indeed, if we go into the docker and ran `cat etc/hosts` we will see a correct entry like below.

```
docker attach pingme 
#press enter
root@pingme:/# cat /etc/hosts

# modified by weave
10.40.0.0	pingme
127.0.0.1	localhost
::1	ip6-localhost ip6-loopback localhost
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
```

Then to ping the container from the second instance of ubuntu do the following:

```
root@2c96895a9a84:/# ping pingme
PING pingme.weave.local (10.40.0.0) 56(84) bytes of data.
64 bytes from pingme.weave.local (10.40.0.0): icmp_seq=1 ttl=64 time=1.33 ms
64 bytes from pingme.weave.local (10.40.0.0): icmp_seq=2 ttl=64 time=1.34 ms
```
You can see that the second instance does not have a hostname `root@2c96895a9a84` while the first yes `root@pingme:/#`. This is obtained by using weave.

## Starting weave with a different ip range in Production
The default IP address allocation range is 10.32.0.0/12. 
But, this range can be used by other networks. For example, Azure nodes lives in a superset of 10.32.0.0/12 IP address range, specifically in the entire class A private IP range (10.0.0.0/8). This can potentially lead to an overlap situation where an azure node ip address conflicts with a weave host IP address.

If you are using weave's IP address allocator, and are not explicitly specifying a range (with -iprange), then you need to force weave to use the old range by specifying --ipalloc-range=192.168.0.0/16 for example.


> The table below show a table range number of addresses.
> This can be used to select a right range of ip addresses to use
> 
Range          | Number of addresses
-------------- | -------------------
10.0.0.0/8     | 16,777,216
172.16.0.0/12  | 1,048,576
192.168.0.0/16 | 65,536

```bash
#host 1
sudo su
weave launch --ipalloc-range 192.168.0.0/16 172.17.8.102

host 2
sudo su
weave launch --ipalloc-range 192.168.0.0/16 172.17.8.101
```

For cloud providers like azure or google we can use also the provided names.

To get the complete list of weave check the [feature section](http://docs.weave.works/weave/latest_release/features.html#dynamic-topologies).

## Conclusions

In this post we showed how to start with weave network from a fresh ubuntu with docker installed. In the following we will show how:

1. weave dns an proxy work behind your containers
2. how use weave to load-balance your services
3. advanced configurations




