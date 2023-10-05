---
layout: post
title: "Probes into Plone and Zope"
date: 2015-03-27 19:31:57 +0200
tags: Python Plone Zope Buildout Monitoring
---

*On all teams, we need to check if Plone turns well. We need some probes to be sure our Plone site runs well*

## Introduction
On all teams, we need to check if Plone turns well. We need some probes to be sure our Plone site runs well !

With Jean-François Roche, we started to have a look on Products.ZNagios. This product allow you to have some probes from Zope, you can ask your instance (live):
- Number of unresolved conflict on Zope
- CPU usage
- DB sizes
- Memory percent
- Uptime of Zope
- ...

You can access to the probes with a thread which listen on Zope  on port 8888 (in this conf). You just have to add zope-conf-additional in your buildout like this:
```
[instance]
...
zope-conf-additional =
  <product-config five.z2monitor>
    bind 0.0.0.0:8888
  </product-config>
```
If you want more information on this, you can see documentation of [five.z2monitor](https://github.com/zopefoundation/five.z2monitor) package.

## collective.monitor
I created this package for adding some probes into Plone. I created probes as Products.ZNagios. We used a zope interface for registering all probes (zc.z3monitor.interfaces.IZ3MonitorPlugin). In this package, I added these probes:
- count users
- count valid users (user logged during 3 last months)
- check if smtp is set up
- last login time of a user
- last time a plone or zope object was modified

## How use it
Adding [collective.monitor ](https://github.com/collective/collective.monitor) in your buildout in eggs and zcml instance section

```
[instance]
...
eggs +=
    ...
    collective.monitor
zcml +=
    ...
    collective.monitor
```
And also adding zope-conf-additional as explain above.

After this little config, you can access to probes with different way

1. bin/instance

After starting instance (bin/instance fg) you can access to probes with
```
./bin/instance monitor dbinfo main
./bin/instance monitor objectcount
./bin/instance monitor stats./bin/instance monitor help
```

2. netcat

After starting instance (bin/instance fg) you can access to probes with
```
echo 'dbinfo main' | nc -i 1 127.0.0.1 8888
```

3. telnet

After starting instance (bin/instance fg) you can access to probes with
```
$ telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
last_modified_zope_object_time
2015/08/11 11:49:48.540729 GMT+2
Connection closed by foreign host.
```

## Conclusion

With this package, you can make stats on your instance.

We use [diamond](https://github.com/python-diamond/Diamond) to collect and put informations from probes on [graphite](https://github.com/graphite-project/graphite-web).

It's very helpful for having state of our infrastructure.