---
title: "m5p3nc3r"
date: 2019-03-13T17:28:09+02:00
description: "Blog posts, technical diary etc. from m5p3nc3r"
---
## Hello

I'm using this website to pull together a number of threads going on in my life at the moment.  I have a number of pet projects that I'm working on and intend to use this site as a location to document what I am doing.  I hope this will help keep me motivated to finish stuff as I once took the [Belbin](www.belbin.com) team profile test, and I came out as a Plant.  If I remember correctly, this means I'm good at starting/creating things, but not very good at finishing them :¬).

## How this site is hosted

I want this site to be an expression of the technology I am interested in, have worked on in the past and am interested in learning.  As a result, I want to achieve the following objectives

* [ ] The site should be hosted on an Arm platform
* [ ] The host for this site should be on my home network
* [ ] The root filesystem for the host should be built with Yocto
* [ ] The site itself should be served from a container

## Hosting on an Arm platform

Why host from an Arm platform?  I have worked on embedded platforms for most of my career.  I also work for [Arm](https://www.arm.com) as software architect in our Automotive and Embedded business line.  Arm and the Arm architecture are my passion, having started my bedroom coding career on Acorn computers including the [BBC Basic](https://en.wikipedia.org/wiki/BBC_BASIC) on the [BBC Micro](https://en.wikipedia.org/wiki/BBC_Micro) and my first taste of the [Arm RISC instruction set](https://en.wikipedia.org/wiki/ARM_architecture#Acorn_RISC_Machine:_ARM2) on the [Acorn Archimedes](https://en.wikipedia.org/wiki/Acorn_Archimedes).

So, I have no choice...  This has to be hosted on an Arm machine.  You can check out the systems I am or have played with in the [boards](/boards) section of the site.

## Hosting on my home network

This is not strictly necessary as there are a number of cloud hosting platforms that I could use for this project.  But, I am interested in understanding how to configure my network to enable this, including how to secure the network.  I'm hoping I don't regret this, but it should be enlightening.

I plan to document my setup here at some point, including how things are configured.  But mainly for me as my memory is not great and I'm using this site as a technical diary.

## Yocto root filesystem

I have been using Yocto on and off since before it was created in Linux Foundation as an evolution of OpenEmbedded.  I was even present at the launch announcement at the Embedded Linux Conference in Cambridge where it was announced.

Yocto is an important tool to understand when working with building a functional root filesystem.  In this case, I will be using the Poky distro to build a minimal system that has just enough functionality to launch a container runtime.

## Containerised applications

I have also been following the development of containers and have been involved in the evolution of multi-architecture container runtimes within the Moby project and the OCI.  I really want to be making use of the cloud-native devops workflows to build and deploy workloads to my embedded platform.  Making use of modern orchestration tools and techniques.  This might seem overkill for a single website, but the idea is that I am using this as a learning and development platform for myself.

## Progress

This is the current TODO list:

* [x] Create a static website based on markdown built using [hugo](https://gohugo.io/)
* [x] Add simple custom responsive theme, this is currently using [bulma](https://bulma.io/)
* [ ] Making use of custom web components using [stencil](https://stenciljs.com/)
* [ ] Build a hosting machine
* [ ] Serve the content
