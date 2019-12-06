+++
draft = false
date = 2014-10-14T22:41:00Z
title = "Service discovering Docker cluster on Digital Ocean"
description = ""
slug = ""
tags = ["docker"]
categories = []
externalLink = ""
series = []
+++
Jeff Lindsay wrote about [“Consul Service Discovery with Docker”](http://progrium.com/blog/2014/08/20/consul-service-discovery-with-docker/) and [Automatic Docker Service Announcement with Registrator](http://progrium.com/blog/2014/09/10/automatic-docker-service-announcement-with-registrator/). Using Docker’s event stream is an elegant solution to finding out which containers are running on a particular host. Despite Jeff’s documentation and videos, I couldn’t get Consul to serve up the correct locations of my services.

Here is a description of what I did to get a service discovering cluster to run on Digital Ocean.

## Goal
I want to have a highly available service running on my Docker cluster, discoverable by using some command line or code. One service should be able to find any number of other services and instances on any host in the system.

## Overview of the solution
Each docker node has a Consul and a registrator container. They could also have some Nginx instances. The first node, here called `docker1`, can for example have one instance of HAProxy running, which will be the entrance to the Nginx cluster. For a production setup, you would typically set this up to not be a single point of failure.

## Set up Digital Ocean
Log in to your Digital Ocean account and start three droplets. Name them `docker1`, `docker2`, and `docker3`. Use the smallest size. Put them in the same region, enable private networking and select the “Docker 1.3.0 on Ubuntu 14.04” application. Last, add your SSH key if you have one configured. None of the Nginx containers are accessible directly from the open internet, only the HAProxy and Consul containers are. Wait for the droplets to start and ssh in to each one.

The `eth0` interface is public, `eth1` is the private network. I’m using this private network for Consul’s internal communication.

## Start Consul
I used `progrium/consul` to start Consul in a Docker container. The `cmd:run` command comes in handy.

First, execute the following command on each node to store the private IP in an environment variable:
```bash
PRIVATE_IP=$(ifconfig eth1 | awk -F ' *|:' '/inet addr/{print $4}')
```
## Node 1
The first host has a slightly different startup incantation:

```bash
$(docker run --rm progrium/consul cmd:run $PRIVATE_IP -d -p 8080:8500)
```
Let’s break this apart. The `cmd:run` command outputs a docker run statement with all the ports linked correctly. This gets wrapped in a subshell. Next comes the IP address Consul should communicate over with other nodes. Last, I’m passing `-d` and `-p 8080:8500` to the run command so the Consul container starts in daemon mode and exposes port `8080` on the host’s public IP.

Note the private IP for node 1, we’ll need this in the next step. Run `ip addr show  eth1` to show the IP.

## Nodes 2 and 3
For the last two nodes we need the private IP address of node 1. On both nodes, run:

```bash
JOIN_IP=<private IP of docker1>
$(docker run --rm progrium/consul cmd:run $PRIVATE_IP::$JOIN_IP -d -p 8080:8500)
```
The two nodes will join the Consul cluster and all will be well.

## Start registrator
When we start a Docker container with exposed ports, we want it to be registered with Consul automatically. For this, we have registrator.

```bash
docker run -v /var/run/docker.sock:/tmp/docker.sock -h $HOSTNAME -d --name registrator progrium/registrator consul://?????:8500
```
Wait… how can the registrator find the Consul container? We could link them using `--link`, we could find out and specify the Consul container’s internal IP address using docker inspect consul. Instead, we’re going to use another elegant solution that’s part of Jeff’s `docker-consul` image. When using the `cmd:run` command, port 53/udp (DNS) gets exposed on the docker engine IP, 172.17.42.1.

_Consul functions as a DNS server so we can find out the location of Consul itself using DNS._

```bash
docker run -v /var/run/docker.sock:/tmp/docker.sock -h $HOSTNAME -d --name registrator --dns 172.17.42.1 progrium/registrator consul://consul.service.consul:8500
```
This works for any container that wants to discover services. Note that [Jeff suggests using DNS exactly like this](https://github.com/progrium/docker-consul#dns). Since it’s worded as a suggestion, I thought it would only be a convenience. Instead, this is one of the keys to make this setup work. Follow Jeff’s advice and set the `DOCKER_OPTS` env variable. For clarity’s sake, I will keep on adding `--dns` to the docker run commands, but it’s not necessary to do that if you set the environment variable correctly.

## Conclusion
Start a bunch of Nginx servers, use `docker run -dP nginx`. Then, run a container that has `dig` installed, for example `docker run --rm -ti --dns 172.17.42.1 remmelt/debian:sid-dig bash`. Request the SRV records: `dig nginx-80.service.consul SRV` and look!

```
; <<>> DiG 9.9.5-4.2-Debian <<>> nginx-80.service.consul SRV
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1638
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 3

;; QUESTION SECTION:
;nginx-80.service.consul.	IN	SRV

;; ANSWER SECTION:
nginx-80.service.consul. 0	IN	SRV	1 1 49168 docker1.node.dc1.consul.
nginx-80.service.consul. 0	IN	SRV	1 1 49166 docker1.node.dc1.consul.
nginx-80.service.consul. 0	IN	SRV	1 1 49164 docker1.node.dc1.consul.

;; ADDITIONAL SECTION:
docker1.node.dc1.consul. 0	IN	A	10.133.240.63
docker1.node.dc1.consul. 0	IN	A	10.133.240.63
docker1.node.dc1.consul. 0	IN	A	10.133.240.63

;; Query time: 13 msec
;; SERVER: 172.17.42.1#53(172.17.42.1)
;; WHEN: Fri Oct 17 22:38:35 UTC 2014
;; MSG SIZE  rcvd: 356
```
Consul responds with a list of nodes running the requested service, including ports, in random order.

Automate this, for example using [Confd](https://github.com/kelseyhightower/confd) or by requesting DNS SRV records in your application, and you have a service discovery!