---
layout: post
title: Docker for Ubiquiti
date: 2019-01-25

---

Over two years since my last post... Wow I'm bad at this!

A lot has changed for me over that time, but thats not for here...  I need to document changes in my home network, just in case things go horribly wrong and I need to remember what it was I did.

I wanted to use Fedora as the host for the services, but the official binaries from Ubiqiti are only available for Debian and Ubuntu.  I played for a while with using virt-manager and boxes to get things up and running, but it kinda felt wrong..   I've been playing with containers for a while now, surely someone has packaged the functions I want already?

The answer to that was of course yes!  Thanks so much to ghe guys who have done the leg-work here.  Lets hope that Ubiquiti support this officially soon, its a so much easier way to deploy the applications!

I'll add more when I can, but for now I'm viewing this as my personal diary..

Taken from <https://hub.docker.com/r/jacobalberty/unifi/>

```bash
docker run --rm --name unifi --init \
        -p 8080:8080 \
        -p 8443:8443 \
        -p 3478:3478/udp \
        -p 10001:10001/udp \
        -e TZ=Europe/London \
        -v /var/lib/unifi:/unifi \
        jacobalberty/unifi:stable
```

Taken from <https://hub.docker.com/r/pducharme/unifi-video-controller>

```bash
docker run \
        --name unifi-video \
        --cap-add SYS_ADMIN \
        --cap-add DAC_READ_SEARCH \
        -p 10001:10001 \
        -p 1935:1935 \
        -p 6666:6666 \
        -p 7080:7080 \
        -p 7442:7442 \
        -p 7443:7443 \
        -p 7444:7444 \
        -p 7445:7445 \
        -p 7446:7446 \
        -p 7447:7447 \
        -v :/var/lib/unifi/video \
        -v :/var/lib/unifi/video/videos \
        -e TZ=Europe/London \
        -e PUID=99 \
        -e PGID=100 \
        -e DEBUG=1 \
        pducharme/unifi-video-controller
```

The next question was how to get the services up and running on system boot using systemd.  Details were taken from <https://container-solutions.com/running-docker-containers-with-systemd/>, but my configuration for the controller ended up looking like this

```bash
[Unit]
Description=unifi Controller (docker)
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
Restart=always
#These start with a - because they are allowed to fail
ExecStartPre=-/usr/bin/docker kill unifi
ExecStartPre=-/usr/bin/docker rm unifi

ExecStart=/usr/bin/docker run --rm --name unifi -p 8080:8080 -p 8443:8443 -p 3478:3478/udp -p 10001:10001/udp -e TZ=\'Europe/London\' -v /var/lib/unifi:/unifi jacobalberty/unifi:stable

ExecStop=-/usr/bin/docker kill unifi

[Install]
WantedBy=multi-user.target
```
