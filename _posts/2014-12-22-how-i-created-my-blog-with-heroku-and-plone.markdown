---
layout: post
title: "How I created my blog with Heroku and Plone"
date: 2014-12-22 20:31:57 +0200
tags: Heroku Python Plone
---

*In this post, I explain how I use Heroku and the [heroku build pack](https://github.com/plone/heroku-buildpack-plone) for the creation of a blog. I have thinking about create a blog with plone and Heroku after the ploneconf in Bristol and the talk of [zupo](https://github.com/zupo) (thanks for his great job).*


So I started... I created a blog because of a friend (a no developer guy) would like to have a blog. And I thought, maybe it's a good idea to try plone with heroku. In this case I tried to make a very easy Plone site with only "Blog Post" object available (I changed my mind after). I created 2 pacakges:

- blog.post
- blog.policy


 And I also create a [buildout](https://github.com/bsuttor/blog.buildout).

Buildout was very easy because of documentation of [heroku build pack](https://github.com/plone/heroku-buildpack-plone), I added a heroku.cfg file on my buildout pakcage. This heroku file extends buildout.cfg with this parts for instance with relstorage :

```
[instance]
recipe = plone.recipe.zope2instance
relative-paths = true
eggs +=
    RelStorage
    psycopg2

rel-storage =
    keep-history false
    blob-dir /tmp/blobcache
    shared-blob-dir false
    type postgresql
    host PG_HOST
    dbname PG_DBNAME
    user PG_USER
    password PG_PASS
 ```

I already have a free account on Heroku, so I simply push my heroku branch and the build pack automaticaly added a postgres plugin, start buildout with heroku.cfg... And it was online.

In my policy, I used and install these packages:

- sc.social.like
- collective.contentrules.yearmonth
- plonetheme.bootstrap
- plone.formwidget.captcha

I still have little work to do as
- change logo
- improve comments
- improve rss feed view (not only title and description into rss feed)
Now you have no more excuse for creating your personnal blog with plone for improving Plone communication and community !