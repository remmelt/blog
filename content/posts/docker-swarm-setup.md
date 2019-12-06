+++
draft = false
date = 2014-12-07T22:17:00Z
title = "Docker Swarm setup"
description = ""
slug = ""
tags = ["docker", "swarm"]
categories = []
externalLink = ""
series = []
+++
_Edit 2015-01-07: Updated article to reflect changes in swarm. Thanks Rael!_


[Docker Swarm](https://github.com/docker/swarm) was announced at the first [European DockerCon](http://europe.dockercon.com/). Swarm is a pluggable cluster manager with a simple scheduler.

It’s currently not super easy to set up, so here is how I did it.

# Background

The `swarm` executable discovers hosts by reading entries from the discovery url, [discovery-stage.hub.docker.com/v1/clusters/TOKEN](https://discovery-stage.hub.docker.com/v1/clusters/TOKEN). Sample output:
```json
["128.199.36.196:2375","128.199.50.146:2375","128.199.32.159:2375"]
```
As you can see, this is just a list of IP/port combinations.

Currently, `swarm` does not need to run on the hosts themselves. The hosts do need to run docker from the master branch, so the local `swarm` can talk to it.

Here is a list of the main steps, I’ll go into more detail later. tl;dr:
1. Create a `swarm token: swarm create`
1. Add hosts to the swarm: `swarm join --discovery=token://TOKEN --addr=host_ip:port`
1. Start the `swarm` manager: `swarm manage --discovery=token://TOKEN --host=swarm_ip:port`
1. Start containers in the swarm: `docker -H swarm_ip:port run -dP nginx`

# The `swarm` application
The `swarm` application does a couple of things. It creates a token, and you can add hosts to the discovery address with it. It also functions as a daemon with a rest API. All of these things can run from your local machine, any of your swarm hosts, or from a container.

Swarm is not distributed as an executable at the moment, so we’ll have to compile it. There are multiple options.

If you want to manage your swarm from OSX, install golang using Homebrew: `brew install go`. Then, get and build the swarm executable by running
```bash
GOPATH=$(pwd) go get github.com/docker/swarm
```
You will find the OSX `swarm` executable under bin/swarm.

If you are running Linux or want to compile for running on Linux using OSX, either install go or use the official go container:
```bash
docker run -v $(pwd):/go/bin --rm golang:1.3.3 go get github.com/docker/swarm
```
This will leave the Linux `swarm` binary in your current working directory.

My favorite option is to just use a container for running swarm. You can use the image that I created in this automated swarm build or roll your own.

# Prepare hosts

The hosts need to run docker version 1.4.0 or higher.

I’m going to run this on Digital Ocean, using the docker image. Start up a new droplet, any size will do. Wait for it to boot, then ssh in and add `DOCKER_OPTS="-H 0.0.0.0:2375"` to `/etc/default/docker`.

Restart the daemon using sudo service docker restart
```bash
ssh root@digitalocean_droplet
service docker stop
echo DOCKER_OPTS=\"-H 0.0.0.0:2375 -H unix://var/run/docker.sock\" >> /etc/default/docker
service docker start
```
Repeat for all the hosts that you want to manage. Done!

# Manager
Now we’re ready to start managing.

## Create a new swarm token
```bash
swarm create
```
will spit out a token. If you're using the swarm image, go
```bash
docker run –rm remmelt/swarm create
```

## Add each of your hosts to the swarm
```bash
swarm join –discovery=token://TOKEN –addr=a.b.c.d:2375
```
This will add address `a.b.c.d:2375` to the swarm. Remember that you can see what is in the swarm by surfing to the [dicovery url](https://discovery-stage.hub.docker.com/v1/clusters/TOKEN).
Running the image? Enter
```bash
docker run –rm remmelt/swarm join –discovery=token://TOKEN –addr=a.b.c.d:2375
```

Note that this command does not currently return. Once the IP has been added to the swarm, you can ctrl-c out. The `join` command does not need to keep running to stay joined.

## Start the management API
```bash
swarm manage –discovery=token://TOKEN –host=0.0.0.0:4243
```
This will start the management API. Image:
```bash
docker run –rm -p 4243:4243 remmelt/swarm manage –discovery=token://XXXX –host=0.0.0.0:4243 ``` (we need the port to communicate with the API).
```
## Start a container
Use the docker binary to start a container as usual, but make sure the `-H` flag points to the host you’re running swarm manage on. In my case, that’s boot2docker.
```bash
docker -H boot2docker:4243 run -dP nginx
```
This will take a while, one of the nodes is downloading the nginx image. The command will return once the container has started.

## List nodes
```bash
swarm list --discovery=token://TOKEN
```
or
```bash
docker run --rm remmelt/swarm list --discovery=token://TOKEN
```
## List containers
```bash
docker -H boot2docker:4243 ps
```
Note the names of the containers, they start with the name of the node they’re running on.

Troubleshooting
If you’re getting a
`tls: oversized record received with length`
error, run `unset DOCKER_TLS_VERIFY` and it will work.

Happy swarming!

