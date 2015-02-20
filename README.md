# docker-dnsmaq
Provides an Docker event listener that dynamically updates a Dnsmasq service for container name resolution

Tested OS:
* CentOS 6

## Background
I was looking for a way to solve a particular issue in my Docker setup - specifically, I needed containers to be able to talk to each other.

This is not a new or unique problem.

But, I needed the containers to be able to talk to each other using any potential FQDN I chose.  For example, maybe I want to talk to a web container as if it were "www.<insert_well_known_website>.com".  The problem here is that most of the solutions out there use some form of Dynamic DNS manipulation - [SkyDock] (https://github.com/crosbymichael/skydock), [SkyDns] (https://github.com/skynetservices/skydns1), etc...  which would normally work, except that I wanted two more things:

* ability to override or create FQDN within my own network domain - and I don't own the DNS servers nor am allowed to update them
* no one on our team knows how to manage DNS (bind) except me, so it has to be simple
* there may be edge cases where I would like to update the entries manually

Based on all this, modifying a hosts file seemed like the perfect solution - except I didn't want to do that by hand and the hosts file is more or less immutable inside of Docker containers.  After looking around, I ran across Kelly Becker's solution for [Docker Service Discovery using Bind] (http://objectiveoriented.com/devops/2014/02/15/docker-io-service-discovery-your-network-and-how-to-make-it-work/) and specifically, a ruby script called [dynamic_ddns] (https://gist.github.com/KellyLSB/4315a0323ed0fe1d79b6#file-docker_ddns) that he wrote to listen for events and make DNS updates using nsupdate.  Dnsmasq allows you to provide an alternative hosts file, which is easily customizable.  All you need to do is tell Dnsmasq where to find the file, update it, and SIGHUP Dnsmasq itself.  Couldn't be simpler.

So, I shamelessly stole Kelly's script and modified it to do what I needed.

## Overview
`docker-dnsmasq` listens for Docker events and does two things:

* on 'start', it discovers the container hostname, domain, IP, and ID, and adds to an alternative hosts file (customizable) as a hosts file entry
* on 'stop' or 'die', it removes the line matching the container ID from the the alternative hosts file
* after any of the three events, it SIGHUP's the Dnsmasq process to pick up the change

## Install/Use

* Install Dnsmsaq and add/update ```addn-hosts=/tmp/docker.dnsmasq``` in /etc/dnsmasq.conf
* Start Dnsmasq
* Put the "docker_dnsmasq" script somewhere and start it up to run in the background, perhaps like ```nohup docker_dnsmasq dd.log &```, which will log to "dd.log"

At this point, you can start/stop/rm containers and you should see `/tmp/docker.dnsmasq` being modified.  You can test it like so:

```
[root@dev tmp]# host docker-tc-agent018 localhost
Using domain server:
Name: localhost
Address: ::1#53
Aliases:

docker-tc-agent018 has address 172.17.0.23
```

The last piece is to make sure your containers use the docker bridge interface for name resolution.  You can add `--dns <server>` to each container startup, or add it to your Docker daemon options.  For me, the bridge interface was `docker0` and the IP was 172.17.41.1.



