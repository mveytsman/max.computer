---
title: Dilettante
project_type: Security stuff
project_dates: 2014
project_summary: messing with maven
---

The main public repository for Java-ecosystem packages is [Maven Central](http://search.maven.org/). I discovered that when you installed Java packages using a tool like maven or ant, they were served unencrypted over HTTP, without any sort of cryptographic verification of their contents. Anyone who has control over a wifi router could trick Java developers into downloading compromised Jars and run arbitrary code on their systems.

I tried asking the company that runs Maven Central nicely to change this, but they didn't budge. So, to prove a point, I wrote [dilettante](https://github.com/mveytsman/dilettante), a man-in-the-middle proxy that would intercept Jars as they are being downloaded and inject cat pictures into them.

[It worked](http://blog.sonatype.com/2014/07/ssl_connectivity_for_central/), and Maven Central now serves all Jars over SSL by default!

![dilettante](/img/dilettante.screenshot.png)
