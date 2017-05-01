---
title: Tinder Finder
project_type: Security stuff
project_dates: 2013
project_summary: tinder + trigonometry
---

I discovered that Tinder would return distances between users with **extremely** high precision. This is a problem because it allows you to deduce the exact location of a Tinder user by measuring their distance to three known points (this is called [trilateration](https://en.wikipedia.org/wiki/Trilateration)).

I made a demo application to geolocate Tinder users in order to demonstrate to Tinder how serious of an issue this was. They fixed the problem, and afterwards I [disclosed](http://blog.includesecurity.com/2014/02/how-i-was-able-to-track-location-of-any.html) it publicly. I got a bit of [press](https://www.bloomberg.com/news/articles/2014-02-19/new-tinder-security-flaw-exposed-users-exact-locations-for-months) for it too!

![tinder finder](/img/tinderfinder.screenshot.png)
