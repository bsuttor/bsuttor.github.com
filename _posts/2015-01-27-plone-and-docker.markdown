---
layout: post
title: "Plone and Docker"
date: 2015-01-27 18:32:57 +0200
tags: Docker Python Plone
---

*In this post, I will try to explain how we put Plone sites into production in our organization ([iMio](https://www.imio.be), Belgium).*

## Introduction
For this process, we used some software as Puppet, Jenkins, but the process we use should be agnostic from these softwares.

Short story: when a push is made on Github, Jenkins builds a new image for Docker, pushes this image into a private Docker registry and updates the Docker image on server.

![CD - simple.png](<img/CD - simple.png>)


## Docker images
We create Docker images with [packer](https://www.packer.io/). We build .deb files with buildout, mr.bob and Jenkins. We create “debian” folder used to .deb files creation with a mr.bob template. We create 3 deb files:
- plone-core-${version}.deb: contains eggs
- plone-zeoserver-${version}.deb: contains zeoserver config files
- plone-instance-${version}.deb: contains instance config files

After creation of deb files, packer uses (and installs) those deb files to create 2 Docker images:
- docker.private-registry.be/plone.zeoserver.be:latest
- docker.private-registry.be/plone.instance.be:latest
We think this is a good way to have good isolation.

Both images are based on a "base IMIO image". Our base image is based on ubuntu image from docker hub. Each image has a size of +/- 530 MB because we have a lot of plone eggs in our buildout/plone site.

You could also create a simple Dockerfile which pulls a github repo and runs buildout to create your Docker image.

Once Packer has built the Docker images, Jenkins pushes them into a private Docker registry.

## Private registry
For this post, I imagine we have a private docker registry in this url: docker.private-registry.be.

We use private registry to store our images.

Our images are created with tag latest and YYYYMMDD-JENKINS_JOB_NUMBER (20150127-97)
We use a private registry for each environment, (staging, production, …) and we copy images between environments. Actually, we automatically update dev and staging environments and when we see there are no problem, we copy images on production.

## Update production
We use fig to orchestrate our docker containers (zeo server must be started before zeo clients). We use a script to update our docker images. This script checks if the currently running docker containers use the latest image. If not, the script downloads the latest image, stops the docker containers which are running, remove them and restart containers from new images (we use upstart scripts to starting docker daemon).

```
cd /fig/directory;fig pull > /dev/null 2>&1
NAME='plone'
REGISTRY_URL='docker.private-registry.be'
INSTANCE="instance"
ZEO="zeo"
INSTANCE_NAME="instance_1"
INSTANCE_IMAGE="$REGISTRY_URL/$INSTANCE"
ZEO_IMAGE="$REGISTRY_URL/$ZEO"

LATEST_INSTANCE_IMAGE_ID=$(docker images | grep $INSTANCE_IMAGE | grep latest | awk '{print $3}')
LATEST_ZEO_IMAGE_ID=$(docker images | grep $ZEO_IMAGE | grep latest | awk '{print $3}')

TAG_INSTANCE_IMAGE_ID=$(docker images | grep $LATEST_INSTANCE_IMAGE_ID | grep -v latest | awk '{print $2}') 
TAG_ZEO_IMAGE_ID=$(docker images | grep $LATEST_ZEO_IMAGE_ID | grep -v latest | awk '{print $2}')

if [ "$TAG_INSTANCE_IMAGE_ID" != "$TAG_ZEO_IMAGE_ID" ];then
    echo "Error: instance and zeo images are no the same tag !" 1>&2
    exit 1
fi
RUNNING=$(docker ps | grep $INSTANCE_NAME | awk '{print $2}')
LATEST="$INSTANCE_IMAGE:$TAG_INSTANCE_IMAGE_ID"
if [ "$RUNNING" != "$LATEST" ];then
    echo "restarting $NAME"
    stop $NAME
    start $NAME
else
    echo "$NAME up to date"
fi
```

## Storage and backup
We use [Docker data containers](https://docs.docker.com/userguide/dockervolumes/#creating-and-mounting-a-data-volume-container) (called storage in our case) for filestorage, blobstorage and backup folders. We start docker container with --volumes-from option. We have to be carefull to NEVER delete a storage container (maybe we have to improve docker for that).

We configure our buildouts to backup all data into var/backups folder and so, we launch docker with --volumes-from and -v options for backup and restore. Thanks to -v options, backups are stored on server and not in Docker. Later, backups are synced to our backup server.

With this zeo docker image, it’s easy to backup, pack and restore zodb. In the futur, we envision using relstorage instead of zeoserver. But currently, there is no DB admin in the company (hint to our boss ?).

## Conclusion
Docker runs great in production !

I intend to follow [docker machine](https://github.com/docker/machine), [docker swarm](https://github.com/docker/swarm/) and docker compose.

Thank you to my colleagues Cédric de Wilde and Jean-François Roche for having worked with me to setup our production Plone into Docker.