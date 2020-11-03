---
layout: post
title: Docker buildx from scratch
---

I was recently at DockerCon 2019 in San Francisco on the Arm booth presenting the beta build extensions (buildx) that enable the docker cli to build multi-architecture container images and automatically hide them behind a fat manifest.  This is a huge step forward in container image creation tooling that will power a true heterogeneous compute future.

On leaving the conference, I thought it would be fun to build the buildx plugin from sources, I hit a few hurdles on the way, so I will document here what I had to do to get it working.

## Step 1: install docker-ce from the test channel

NOTE: This is only needed if we are not using docker 19.03.x or above

```bash
# I am running on Fedora 30 here - your mileage may vary :¬)
# Remove any old version of docker
dnf remove docker

# Install the beta version of docker from the test repository
wget https://test.docker.com > docker.sh
sh docker.sh
sudo usermod -aG docker $USER
sudo service docker start

# You will probably need to log out and back in again to ensure
# you are recognised as a member of the docker group
docker version
# this should now report we are running a beta version of docker
```

## Step 2 - build the buildx plugin from source

```bash
# Download the buildx sources
git clone https://github.com/docker/buildx && cd buildx
# Follow the instructions for building with Docker 18.09+
# as we don't yet have buildx available
make install
```

## Step 3 - the magic bit to get qemu binfmt_misc working properly

NOTE: This step assume that you already have qemu-user installed

```bash
# This wires up qemu properly for use
docker run --privileged linuxkit/binfmt:v0.7
```

## Step 4 - Create a build context

```bash
# Create a build context
docker buildx create builder
docker buildx use builder
# Initialise the build context, this will create a container image
# running buildkit
docker buildx inspect --bootstrap
```

## Step 5 - test

Create a simple test application that simply outputs the architecture we are executing on.

```bash
#include <stdio.h>
#ifndef ARCH
#define ARCH "Undefined"
#endif

int main() {
  printf("Hello, my architecture is %s\n", ARCH);
}
```

Then create a simple Dockerfile to build the images. Lets make it fun and use a multi-stage Dockerfile.

```bash
FROM alpine AS builder
RUN apk add build-base
WORKDIR /home
COPY hello.c .
RUN gss "-DARCH=\"`uname -a`\"" hello.c -o hello

FROM alpine
WORKDIR /home
COPY --from=builder /home/hello .
ENTRYPOINT ["./hello"]
```

Now lets go ahead and build it for multiple architectures and push to hub.docker.com. This assumes you are already authenticated with hub.

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t m5p3nc3r/hello . --push
# We can no check the generated manifest to see that
# two images are available
docker buildx imagetools inspect m5p3nc3r/hello
# You can then execute any of the images, thanks to the
# magic of qemu and binfmt_misc
docker run m5p3nc3r/hello:latest@sha256 <the SHA from the manifest>
```

## Conclusion

We have successfully created two binary container images, pushed them to hub, created a manifest that will now allow a 'docker pull' to work from both x86 and arm64 hosts.  Cool!
