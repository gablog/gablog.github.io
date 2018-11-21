---
published: false
---
Today we will be diving into Docker layers, how they work and why they are important to Docker and it's container system

![Docker Layers Graphic](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)

## Introduction
Even if you've never heard of Docker layers before, chances are you've come across them if you've ever used Docker or Docker-Compose.

Whenever you first pull or create an image from DockerHub, you'll usually see this 
```
$ docker pull registry
Using default tag: latest
latest: Pulling from library/registry
796ea0cf9cc01: Pull complete
70ce567423403: Pull complete
77ee34d1a7fa4d: Pull complete
34efa8923dd: Pull complete
a8932ef3: Pull complete
Digest: sha256:a3551c42252533223fdc927c05dfe24633216ca363d06c2
Status: Downloaded newer image for registry:latest
```

During this process you have to wait for each pull to complete before going on to the next. Each of these "pulls" with its unique hexadecimal hash are pulling a individual layer of a docker image.

This is why when on subsequent pulls of an image, these layers seem to be cached for you.

## How they work with Docker Images

As seen in the image below, each Docker image is comprised of multiple layers. In this case our Docker container contains a Ubuntu base layer, a Ubuntu update layer, an Apache layer and a customer user defined layer

![](https://images.techhive.com/images/article/2016/06/docker-image-layers-100664051-large.idge.png)

Docker containers are just running instantiations of an image, so they too also have layers. However unlike  images, containers aren't read-only, which allows any runtime changes to update the top user-defined layer as the container runs. This allows us to update code and create Docker volumes, which allow for us to connect our host machine's filesystem with that of our container's.

To make use of these layers in one single image, Docker uses a union file system to combine multiple image layers together. This union filsesystem llows files and directories of other filesystems to be overlaid, creating a single coherent filesystem. This important fact is what allows volumes to work. Without a union filesytem, easily hot-swapping a external file system wouldn't be possible. 

## How are they formed?

A layer is created by every command you specify (i.e. `FROM`, `COPY`, `RUN`) in your Dockerfile. This allows  layers to functional similar to git, in the sense that  each layer is a diff that only contains information about what it changes in the image's filesystem. This layer system allows docker to cache commonly used base images (such as Ubuntu or Alpine Linux), allowing for faster compilation of Docker images and an increase in storage efficiency when storing multiple containers.

Docker layers also help when building images, due to the intermediate state of each layer. When rebuilding a Docker image, only the layers that are changed are required to be rebuilt, significantly decreasing overall build time.