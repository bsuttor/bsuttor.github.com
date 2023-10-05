---
layout: post
title: "New recipe, collective.recipe.buildoutcache"
date: 2015-07-01 21:31:57 +0200
tags: Python Plone Buildout Ci Jenkins Docker
---

*This recipe generate a buildout-cache archive. We use pre-generated buildout-cache folder for speed up buildout duration. The archive contains one single buildout-cache folder*

## Introduction
This recipe generate a buildout-cache archive. We use pre-generated buildout-cache folder for speed up buildout duration. The archive contains one single buildout-cache folder. In this folder, there are 2 folders:

- eggs: contains all eggs use by your buildout except eggs which have to be compiled.
- downloads: contains zip eggs which must be compiled (as AccessControl, lxml, Pillow, ZODB, ...)

Before starting a buildout, we download and extract buildout-cache and use it on our buildout. We add eggs-directory and download-cache parameters on buildout section like this:

```
[buildout]

eggs-directory = buildout-cache/eggs
download-cache = buildout-cache/downloads
```

## Use case
In our organization, we have a Jenkins server. We created a Jenkins job which generate buildout-cache.tar.gz2 every night and push it into a file server.

We also use Docker, our Dockerfiles download and untar buildout-cache before starting buildout, so creation of docker image became very faster !

## How it works
Simply, you have to add an parts with this recipe on your buildout project.

Like this:

```
[buildout]
parts =
    ...
    makebuildoutcachetargz

[makebuildoutcachetargz]
recipe = collective.recipe.buildoutcache
```

You can use some parameters for changing name of your archive, use another working directory than ./tmp or use another buildout file than buildout.cfg for eggs downloads, See https://github.com/collective/collective.recipe.buildoutcache.

For recipe installation you can make this command line:
```
./bin/buildout install makebuildoutcachetargz
```

And start recipe script:
```
./bin/makebuildoutcachetargz
```

## Conclusion
Use [collective.recipe.buildoutcache](https://pypi.python.org/pypi/collective.recipe.buildoutcache) and decrease time lost with your buildout ;-)