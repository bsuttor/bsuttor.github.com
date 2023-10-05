---
layout: post
title: "collective.preventactions"
date: 2016-09-19 21:16:11 +0200
tags: Python Plone
---
*Sometimes we have some folders or some documents which should have not been deleted. But our customers/users have deleted them and it occurs some errors on application. So we decided to create a new package to prevent delete or/and move actions.*

Sometimes we have some folders or some documents which should have not been deleted.

But our customers/users have deleted them and it occurs some errors on application.

So we decided to create a new package to prevent delete or/and move actions.


## How it works
We add marker interface to objects which can not be deleted or renamed (= moved).

We subscribe all IItem objects to `OFS.interfaces.IObjectWillBeRemovedEvent` and `OFS.interfaces.IObjectWillBeMovedEvent`

When one of these events is received, and object is marked as not deleted or not renamed, we [raised an exception](https://github.com/collective/collective.preventactions/blob/master/src/collective/preventactions/prevent.py) and object is not deleted or moved.




> In the futur, we expect to add a dashboard to have a view of all contents with these markers interfaces to easily use it.


You can also set some contents not deleteable (for example) as this in your setuphandler :
```python
from collective.preventactions.interfaces import IPreventDelete
from plone import api
from zope.interface import alsoProvides


def post_install(context):
    obj = api.content.get('/Plone/content-not-deleteable')
    alsoProvides(obj, IPreventDelete)
```

Now you can have a look at [source code of package](https://github.com/collective/collective.preventactions) and try it.