---
layout: post
title: "How I made my wedding site"
date: 2015-03-27 14:31:57 +0200
tags: Python Django Pyramid Mezzanine Cartridge Plone
---

*I decide to make a website for my wedding with a list of gift, my honneymoon, presentation of my witnesses and so on.*

I was looking for a litlle CMS with a "list of gift" (an online shopping) which can be installed on cheap and reliable hosting (And it's when I loose Plone)

## Pyramid vs Django
I started looking on Pyramid (because I'm a Plone/Zope dev). I thought [Kotti](http://kotti.pylonsproject.org/), but I didn't find a way to make easily gift, and I thougt project looks cool, but it'was maybe a little young for my kind of requirements. I didn't find good solution on pyramid for a wedding list.

Such as I have some exprience in [Django](https://www.djangoproject.com/), And in my daily work, we started intereset on [Geonode](http://geonode.org/) for GIS project.

-> I started looking on Django !

## Django CMS vs Mezzanine
[Django CMS](https://www.django-cms.org/) and [Django CMS e-commerce plugin](https://www.django-cms.org/en/e-commerce/). But it seems this project is a almost dead ? [Last commit on github](https://github.com/divio/django-shop/commits/master) make me septic.

With little search, I found [Mezzanine](http://mezzanine.jupo.org/docs/index.html) and [Cartridge](http://cartridge.jupo.org/). I try it and It seems perfect for my porject, So I choose it !

## Hosting
My first choose was OVH, because it's very cheap (5€ / month). But with little search, it is almost impossible to create a complex Django site (by complex, I mean a "Mezzanine" Django site, and it's not very complex). I pursued my searching... And I found Webfaction. They have local pythons, postgres, 600Go data for 10 € / month. It looks perfect for me, except they do not manage domain name directly. So I host my wedding site on webfaction and my domain name on OVH.1

Maybe I could made an heroku Django website, but I was little affraid about complexity.



Next step is to create an online shop with Kotti or with Pyramid !