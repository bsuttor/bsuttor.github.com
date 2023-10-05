---
layout: post
title: "New package: collective.geo.faceted"
date: 2016-09-07 21:16:11 +0200
tags: Python Plone Map Leaflet Javascript
---

*We use collective.geo suite for geolocatisation for some of our projects. We also use eea.facetednavigation to easily find content into our website or application. So we decided to add a map view for eea.facetednavigation and we created collective.geo.faceted.*

## Why did we create collective.geo.faceted ?
We use collective.geo suite for geolocatisation for some of our projects. We also use eea.facetednavigation to easily find content into our website or application.

So we decided to add a map view for eea.facetednavigation and we created collective.geo.faceted.

## How it works
### Leaflet
We prefer to use Leaflet than OpenLayers, because it seems easier to use for us.

So we decided to use collective.geo.leaflet machinery for map creation.

### GeoJson
We use geojson standard to add points on map. It is a famous standard used to geo content.

The view created for "faceted" will simply update the geojson and this geojson will also update the map. For the generation of geojson, we extended collective.geo.json view.

### Viewlet
Map is added into a viewlet dedicated to the faceted view. We choose to use a viewlet out of 'content-core slot'  (content-core slot is used by faceted to update automatically the  contents). Indeed each technology (faceted and map) uses singular  javascript , it seems better to not mix both technology.

Plone objects are on map  and they are updated thanks to these lines of code:

```javascript
jQuery(document).ready(function() {
  jQuery(Faceted.Events).bind(
    Faceted.Events.AJAX_QUERY_SUCCESS,
    update_map
  );
});
```

This code fetches "faceted" events when "faceted" has modifed its criterias and used update_map javascript function to update new geojson on map.

Image is better than words:
![collective geo faceted image](https://raw.githubusercontent.com/collective/collective.geo.faceted/master/docs/screenshot.png)


## Future
This package is tested for Plone 4 with plone.app.contenttypes used as default Plone content types.

Maybe in future we should add a profile like plone5  e.g. [example.p4p5](https://github.com/collective/example.p4p5) or create a branch plone4 on github and make master branch used to plone5.

