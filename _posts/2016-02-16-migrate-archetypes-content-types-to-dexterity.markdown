---
layout: post
title: "Migrate Archetypes content types to Dexterity"
date: 2016-02-16 20:16:11 +0200
tags: Python Plone
---

*By this migartion, I had 3 goals: make my first step for migration to Plone 5, use multiligual site with Plone 4 and plone.app.multilingual, use plone.app.event (for recurence, no end event, ...).*

By this migartion, I had 3 goals:

- make my first step for migration to Plone 5
- use multiligual site with Plone 4 and plone.app.multilingual
- use plone.app.event (for recurence, no end event, ...).

I also don't want to make big visual changes for my clients. So I decided to not use plone.app.widgets at this moment. I prefere use it with Plone 5. So I will pin plone.app.event 1.1.x. Indeed newer versions of plone.app.event use plone.app.widgets.

For this migration I have to done 2 'steps'. I made a 'custom migration' and I had a profile with an upgrade step. I will explain that below.

For migration, I first have install and pin some packages in my buildout:

- Add plone.app.contenttypes to my buildout (with 1.1.x version/branch)
- Pin plone.app.event to 1.1.x version/branch
- Pin plone.outputfilters 2.1.2 for this problem

## Creation of a new profile

I choose to add new profile called 'migratetodx' for making migration. I prefere use a new profile instead of a upgrade step but all migration can be used into a upgrade step.

So I started with creating new profile like this:

```xml
<genericsetup:registerProfile
    name="migratetodx"
    title="cpskin.migration: migrate at to dx"
    directory="profiles/migratetodx"
    description="Updates CPSkin to dexterity"
    for="Products.CMFPlone.interfaces.IPloneSiteRoot"
    provides="Products.GenericSetup.interfaces.EXTENSION"
    />
```

Created folder profiles/migratetodx and added a metadata.xml file like this :

```xml
<?xml version="1.0"?>
<metadata>
  <version>1</version>
  <dependencies>
    <dependency>profile-plone.app.contenttypes:default</dependency>
  </dependencies>
</metadata>
```

## Add behaviors
### Lead image
For lead image, all job is already done in plone.app.contenttypes. I just had to add behavior for types you would like to migrate.

So in my profile, I added files like profiles/migrate/types/Folder.xml

```xml
<?xml version="1.0"?>
<object name="Folder">
  <property name="behaviors" purge="false">
    <element value="plone.app.contenttypes.behaviors.leadimage.ILeadImage"/>
  </property>
</object>
```

### Other collective packages

I added others packages from collective: Collective.geo.*, collective.plonetruegallery and eea.facetednavigation.

So I added other behaviors for my folder type into Folder.xml:

```xml
<?xml version="1.0"?>
<object name="Folder">
  <property name="behaviors" purge="false">
    <element value="plone.app.contenttypes.behaviors.leadimage.ILeadImage"/>
    <element value="collective.geo.behaviour.interfaces.ICoordinates" />
    <element value="eea.facetednavigation.subtypes.interfaces.IPossibleFacetedNavigable"/>
    <element value="collective.plonetruegallery.interfaces.IGallery"/>
  </property>
</object>
```

### Add import step

I created a profiles/migratetodx/import_steps.xml

```xml
<?xml version="1.0"?>
<import-steps>
  <import-step id="cpskin.migration.migratetodx"
               handler="cpskin.migration.migrate.migratetodx"
               title="cpskin.migration: import step for migration">
    <dependency step="typeinfo" />
  </import-step>
</import-steps>
```

And I use migrate.py for preparing migration, starting migration (with migration view) and fixing image scales.

### Problems with memoize

We use caching for our sites and applications. And during migration, I saw than I had some problems with the cache and with plone.memoize. We decide to use an empty plone.memoize cache and keep this cache empty with this code into import step.

In my migrate.py file, I used this code:

```python
from plone import api
from zope.annotation.interfaces import IAnnotations


def migratetodx(context):
    if context.readDataFile('cpskin.migration-migratetodx.txt') is None:
        return
    portal = api.portal.get()
    request = getattr(portal, 'REQUEST', None)
    class EmptyMemoize(dict):

        def __setitem__(self, key, value):
            pass

    annotations = IAnnotations(request)
    annotations['plone.memoize'] = EmptyMemoize()
```

### Fix image scale

We also have to fix new way to get image scale, I was inspared by  [this code](https://gist.github.com/thet/3de0abd7d18e6edc87dd). It get all richtext content and check if it needs to change image_[preview] to @@images/image/[preview].

I also use this code for getting all portlets static and update it.
```python
from zope.component import getMultiAdapter
from zope.component import getUtility
from plone.portlets.interfaces import IPortletManager
from plone.portlets.interfaces import IPortletAssignmentMapping

def image_scale_fixer(text):
    if text:
        for old, new in IMAGE_SCALE_MAP.items():
            # replace plone.app.imaging old scale names with new ones
            text = text.replace(
                '@@images/image/{0}'.format(old),
                '@@images/image/{0}'.format(new)
            )
            # replace AT traversing scales
            text = text.replace(
                'image_{0}'.format(old),
                '@@images/image/{0}'.format(new)
            )
    return text

def fix_portlets_image_scales(obj):
    managers = [u'plone.leftcolumn', u'plone.rightcolumn']
    for manager in managers:
        column = getUtility(IPortletManager, manager)
        mappings = getMultiAdapter((obj, column), IPortletAssignmentMapping)
        for key, assignment in mappings.items():
            # skip possibly broken portlets here
            if not hasattr(assignment, '__Broken_state__'):
                if getattr(assignment, 'text', None):
                    clean_text = image_scale_fixer(assignment.text)
                    assignment.text = clean_text
            else:
                logger.warn(u'skipping broken portlet assignment {0} '
                            'for manager {1}'.format(key, manager))
```

## Custom migration
### Migrate extended archetypes field

I migrated an extend archetype filed named 'hiddentags'. For that is use ICustomMigrator adapter. I added this line on my configure.zcml:

```
<adapter name="mymigrator" factory=".migrate.MyMigrator" />
```

And I created a class MyMigrator with a "migrate"method in my file migrate.py

```python
from plone.app.contenttypes.migration.migration import ICustomMigrator
from zope.component import adapter
from zope.interface import implementer
from zope.interface import Interface


@implementer(ICustomMigrator)
@adapter(Interface)
class MyMigrator(object):

    def __init__(self, context):
        self.context = context

    def migrate(self, old, new):
        # hiddenTags
        if getattr(old, 'hiddenTags', None):
            new.hiddenTags = old.hiddenTags
```

### Migrate marker interfaces

I had some marker interfaces on our content, in this snippet, I will show how I migrated eea.facetednavigation marker :

```python
from eea.facetednavigation.settings.interfaces import IDisableSmartFacets
from eea.facetednavigation.settings.interfaces import IHidePloneLeftColumn
from eea.facetednavigation.settings.interfaces import IHidePloneRightColumn
from eea.facetednavigation.subtypes.interfaces import IFacetedNavigable
from eea.facetednavigation.subtypes.interfaces import IFacetedWrapper

interfaces = [
    IFacetedNavigable,
    IDisableSmartFacets,
    IHidePloneLeftColumn,
    IHidePloneRightColumn,
    IFacetedWrapper,
]
for interface in interfaces:
    if interface.providedBy(old):
        alsoProvides(new, interface)
```

### Migrate faceted criteria

In our site, we use faceted navigation. and we have to migrate criteria for all our faceted view. I have done into custom migration with that code:

```python
if IFacetedNavigable.providedBy(old):
    criteria = Criteria(new)
    criteria._update(ICriteria(old).criteria)
```

### Migrate object with coordinates

Again in migrate method, I had this snippet

```python
from collective.geo.behaviour.behaviour import Coordinates

old_coord = Coordinates(old).coordinates
new_coord = Coordinates(new)
new_coord.coordinates = old_coord
```

### Setting time-zone to new event

When I wrote this code, there is still a bug into plone.app.event 1.1.0 and plone.app.contenttypes 1.1.0. Time-zone is not set on each event during migration, I forced id:

```python
from plone.app.event.dx.interfaces import IDXEvent

if IDXEvent.providedBy(new):
    new.timezone = timezone
```

## Conclusion
I learnt a lot of Plone migration all along my work.Each migration depend on plugins you use,

And [this is link for my 'import step' script](https://github.com/IMIO/cpskin.migration/blob/master/cpskin/migration/migrate.py). I hope you can use some piece of code of it.

