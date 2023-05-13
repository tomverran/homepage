---
title: WordPress is quietly amazing
date: 2023-05-13
---

Recently I've updated two websites I started hosting for local organisations in 2009. 
I certainly didn't imagine for one second that fourteen years later I'd still be hosting them and 
one of them was using a WordPress theme I wrote in 2011.

<!--more-->

I hadn't touched either of the websites for many years so it was with some trepidation
that I began the process of updating everything and moving them to a new `t4g.micro` ARM AWS instance. 
I was pretty sure I'd find out that I'd need to rewrite the 2011 theme from scratch,
after all it consisted of some hacky CSS I layered on top of the WordPress [Twenty Eleven](https://en-gb.wordpress.org/themes/twentyeleven/) theme.

I was amazed, therefore, when after changing a lot of version numbers I was able to re-run the Ansible scripts I'd created (thank you, past me!) 
and it all.. just... worked! I was even more astonished when I found out that the last update made to the Twenty Eleven 
theme was in March 2023. The theme itself has had more than thirty updates since I first built on top of it, and each 
of them has been 100% compatible with my alterations.

For context, between 2011 and now we've seen:

- Windows 8 and 8.1 being released and then having support dropped.
- React being created and _eighteen_ major versions packed with breaking changes being released.
- Twelve major iOS releases, eight of which are no longer receiving security updates.

I can think of very few companies that still maintain their products after more than a decade, and even fewer 
that do so for free, while maintaining absolutely perfect compatibility (at least for me). 
Even the EC2 instance I was moving the site off had lost its ability to communicate with the AWS API
 thanks to some breaking changes made by Amazon.

It isn't just me being lucky with the theme either, my Ansible scripts include a `wp-config.php`
(WordPress' main configuration file) that I generated long ago from an ancient installation and it still works flawlessly.

I was always a bit snobbish about WordPress - especially when I was a full-time PHP developer - given its 
legendarily unfashionable codebase and [extremely dangerous](https://wpscan.com/plugins) plugin ecosystem 
but this experience has completely reversed my viewpoint. 

While I'd still steer _very_ clear of significant customisations (and most third party plugins) I now think of 
WordPress as a pretty viable platform to build a long-term site on - maybe moreso than the majority of other things 
out there. I can't see any site built on a modern tech stack still being effortlessly supported after more than a 
decade given the incredibly low lifespan of quite foundational JS libraries.

Even WordPress' codebase is coming back into fashion now that JavaScript developers 
are starting to write about [_advanced new techiques_](https://medium.com/@ggonchar/api-less-architecture-with-react-server-component-e8d37c3b1a06)
that let them merrily mix all of their database code into their HTML. 🙃
