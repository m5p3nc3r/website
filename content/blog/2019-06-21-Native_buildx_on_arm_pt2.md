---
layout: post
title: Creating a native Arm builder for Docker buildx Part 2
---

After getting the USB gadget configured in the previous post, its time to install docker-ce on the device.  Because we have an active network connection and docker-ce is an [officially supported platform](https://docs.docker.com/install/), this is a simple process of following the official installation documentation.

```bash
curl -fsSL https://test.docker.com -o test-docker.sh
sh test-docker.sh
```

Docker should now be installed and running, but you won't be able to contact the docker daemon.  This is covered in [this post](https://docs.docker.com/install/linux/linux-postinstall/).

```bash
# Add your user to the 'docker' group
sudo usermod -aG docker $USER
# You now need to log out and log back in again for this to take effect
```

## Enabling remote access to the docker daemon

Next you need to allow remote access to the docker daemon.  This will depend on the distro you are using, but mine is systemd based, so I follow the instructions in the section [configure where the docker daemon listens for connections](https://docs.docker.com/install/linux/linux-postinstall/#configure-where-the-docker-daemon-listens-for-connections).  I add the endpoint  ```-H tcp://0.0.0.0:2375```.  This will allow connection from any configured network interface on port 2375.

So my ```ExecStart``` line now looks like

```bash
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
```

That should be it for the device configuration.  I will loop back in a later post to try and make all of this configuration automatic, because at the moment the USB gadget network interface won't come online automatically.  But we are still very much in the proof-of-concept phase!

## Configuring buildx on the laptop

You need to be using the latest _edge_ release of docker.  In this post, I am using Docker-ce version 19.03.0-rc2, with the buildx version I created in my previous post [Docker Buildx from scratch](../Docker_buildx_from_scratch/).

There are many good resources on the web for getting started with buildx, I would recommend reviewing [some](https://engineering.docker.com/2019/04/multi-arch-images/) [of](https://community.arm.com/developer/tools-software/tools/b/tools-software-ides-blog/posts/getting-started-with-docker-for-arm-on-linux) [them](http://collabnix.com/building-arm-based-docker-images-on-docker-desktop-made-possible-using-buildx/).  Or if you are that way inclined, I would recommend reading the documentation at the [buildx github](https://github.com/docker/buildx).

We need to create a builder instance for buildx to use when targeting our USB gadget.

````bash
# Create the buildx context
docker context create --docker "host=tcp://192.168.0.2:2375" armbuilder
# select the newly created context
docker context use usbgadget
# Check everything is working by inspecting the builder context
docker buildx inspect
````

The output for me looks like this

````bash
Name:   default
Driver: docker

Nodes:
Name:      default
Endpoint:  armbuilder
Status:    running
Platforms: linux/arm64, linux/arm/v7, linux/arm/v6
````

Now I can re-run the example code I put up in [Docker Buildx from scratch](../Docker_buildx_from_scratch/).  But for now, I only have arm builders configured in this context, so I issue the command

```bash
docker buildx build --platform linux/arm64 -t m5p3nc3r/hello .
```

And the build will take place on the USB gadget :¬).  We can check that the build worked, assuming you have qemu configured correctly on your host PC by running

```bash
docker run m5p3nc3r/hello
Hello, my architecture is Linux buildkitsandbox 3.14.79-116 #1 SMP PREEMPT Tue Sep 26 01:19:06 BRT 2017 aarch64 Linux
```

## Using the buildkit container for building images

This setup is not currently using the buildkit container to execute the builds in, this means that we are missing some core functionality like the ability to be able to build the multi-architecture manifest file and push multiple images to hub.docker.com.  I have some issues to resolve getting this working..

To try using the buildkit container, we setup the build context like this

```bash
# Create the buildkit builder referencing our USB gadget
docker buildx create --name armbuildkit --driver docker-container tcp://192.168.0.2:2375
# Select the buildx context
docker  buildx use armbuildkit
# Inspect the buildx context (note --bootstrap to ensure the remote context is running)
docker buildx inspect --bootstrap
```

Where I now see the output

```bash
Name:   armbuildkit
Driver: docker-container

Nodes:
Name:      armbuildkit0
Endpoint:  tcp://192.168.0.2:2375
Status:    running
Platforms: linux/arm64, linux/arm/v7, linux/arm/v6
```

But now when I try the build, I get the following error

```bash
 => [internal] load build context                                                                                  1.8s
 => => transferring context: 227B                                                                                 0.0s
 => ERROR [builder 2/5] RUN apk add build-base                                                                    1.0s
 => [stage-1 2/3] WORKDIR /home                                                                                   1.1s
------
> [builder 2/5] RUN apk add build-base:
#6 0.409 nsenter: could not ensure we are a cloned binary: Operation not supported
#6 0.976 container_linux.go:345: starting container process caused "process_linux.go:299: copying bootstrap data to pipe caused \"write init-p: broken pipe\""
------
failed to solve: rpc error: code = Unknown desc = failed to solve with frontend dockerfile.v0: failed to build LLB: executor failed running [/bin/sh -c apk add build-base]: exit code: 1
```

But I am going to leave solving this for another day.  The buildkit integration is in beta at the moment, so this may not be a problem with my setup but the code.  I'll file a bug report and see if I can get help getting this resolved.

## Conclusion

Almost there!  A successful first build of an arm image on an arm host :¬).

As for the next steps, I am going to spend the next session making the setup more robust and easier to work with.  Also need to look into what the issue with buildkit is so that we can get the docker-container driver working properly.
