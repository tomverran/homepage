---
title: Tube Status
techs: [Scala, Heroku, AWS]
links: {Github: https://github.com/tomverran/tube-status}
headless: true
date: 2017-07-01
---

The story of my tube status website began in July 2017 when on a baking hot summers day I descended into Leicester Square tube station
only to discover that all the Piccadilly Line trains were running up to 8 minutes apart which, trust me, is not good news.

I was shocked to discover that TFL described the situation as a good service on their tube status app so attempted to create
an alternative app that used the TFL API and AWS CloudWatch to measure the average duration between trains at certain stations and decide how good the service was.

The problem with this idea turned out to be that many tube stations didn't provide very accurate countdowns (looking at you, Notting Hill) so
I ended up giving up on this idea fairly quickly.
