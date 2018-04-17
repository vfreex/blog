---
layout: post
title:  "Customize Docker network"
date:   2015-11-04 17:21:00 +0800
categories:  linux docker
tags: technology container
---
[Docker][] is an awesome software which makes use of Linux container
to package dependencies for an application and isolate its runtime environment from the host machine.
By default, Docker will set up an isolated private network and adopt [Network address translation][] (NAT) technology
for forwarding incoming traffic to corresponding container. But sometimes we want to set up our own Docker network.

A few days ago, I was going to run several separated [Bind][] DNS services on a single host.
Some of them are recursive servers while others are authoritative servers.
Traditionally, we need to bind these services to different addresses on the same host,
or create virtual machines for every Bind instance.

I chose Docker for this kind of deployment. The difficulty is how to expose the whole network of these containers
to the outside network instead of using NAT technology, because it is very hard to maintain so many IP addresses on a single host machine.
We can use `--net=none` option when starting a container to tell Docker not to configure its network automatically, which leaves us free to configure our own network.

Considering I have a host machine and 4 Docker containers. The host machine's physical interface connects to a router that connects to an intranet.

```
[gateway] eth3: 10.3.0.1/24  <--> [host machine] eth0: 10.3.0.2/24 / docker0: 10.3.16.1/20 <--> [containers] eth0: 10.3.16.11/20-10.3.16.14/20
```

The host machine has 2 IP addresses. One is `10.3.0.2/24`, which connects to a gateway interface `eth3` with IP address `10.3.0.1/24` through its physical interface `eth0`.
The other is `10.3.16.1/20`, which is assigned to the Docker Ethernet bridge interface `docker0`. The four Docker containers have addresses from `10.3.16.11/20` to `10.3.16.14/20`.

By default, Docker configures bridge interface `docker0` with IPv4 address `172.17.42.1/16`. We need to change it to `10.3.16.1/20` in this deployment.
To supply a different IP address for `docker0`, We need to edit Docker upstart configuration file. This file is usually located at `/etc/sysconfig/docker` for Fedora/RHEL/CentOS or `/etc/default/docker` for Debian/Ubuntu.
Here is a resonable configuration:

{% highlight bash %}
# /etc/sysconfig/docker
# Modify these options if you want to change the way the docker daemon runs
OPTIONS='--bip=10.3.16.1/20 --iptables=false'
# ...
{% endhighlight %}

In this file, the `--bip=10.3.16.1/20` option tells Docker to set up bridge interface with IPv4 address `10.3.16.1/20`, and `--iptables=false` to disallow Docker to insert iptables rules automatically.

Restart your Docker service, then start a container:

{% highlight bash %}
$ systemctl restart docker # systemd OS
$ service docker restart # other OS
$ docker run -ti --rm --net=none --name my_container centos:7 /bin/bash # start a container
{% endhighlight %}

This will start a CentOS 7 container named `my_container`. Option `--net=none` tells Docker not to set up container network.
Issuing `ip addr` in this container will find that there are no Ethernet interfaces configured:

{% highlight bash %}
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
{% endhighlight %}

As Docker uses Linux container to isolate environment, we should set up a container network through the host machine.
These instructions are adapted from official [Docker documentation][].

Firstly, use `docker inspect` command to find out the container process ID:

{% highlight bash %}
{% raw %}
PID=`docker inspect -f '{{.State.Pid}}' my_container`
{% endraw %}
{% endhighlight %}

Then, create a peer of virtual network interfaces `A` and `B`, then bind the A end to the bridge, and bring it up:

{% highlight bash %}
$ ip link add A type veth peer name B
$ brctl addif docker0 A
$ ip link set A up
{% endhighlight %}

Next, place B inside the container's network namespace, and rename to eth0, and activate it with IP `10.3.16.11/20`.
Follow similar steps to configure networks for other containers by replacing IP addresses with `10.3.16.12/20` - `10.3.16.14/20`.

{% highlight bash %}
$ ip link set B netns $PID
$ ip netns exec $PID ip link set dev B name eth0
$ ip netns exec $PID ip link set eth0 address 12:34:56:78:9a:bc
$ ip netns exec $PID ip link set eth0 up
$ ip netns exec $PID ip addr add 10.3.16.11/20 dev eth0
$ ip netns exec $PID ip route add default via 10.3.16.1
{% endhighlight %}

Lastly, add a static route on your gateway to forward address range `10.3.16.0/24` to your host machine:

{% highlight bash %}
$ ip route add 10.3.16.0/20 via 10.3.0.2 dev eth3
{% endhighlight %}

In addition, I wrote a script called [Dockernet][] in order to finish these steps in a very simple way. You can check out my code if you like.

[Docker]: https://www.docker.com
[Network address translation]: https://en.wikipedia.org/wiki/Network_address_translation
[Bind]: https://www.isc.org/downloads/bind/
[Docker documentation]: https://docs.docker.com/engine/userguide/networking/
[Dockernet]: https://github.com/vfreex/dockernet
