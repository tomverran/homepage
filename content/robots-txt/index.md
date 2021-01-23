---
title: Robots.txt parser
headless: true
techs: [PHP]
links: 
  GitHub: https://github.com/tomverran/robots,
  Packagist: https://packagist.org/packages/tomverran/robots-txt-checker
date: 2014-04-14
---

In 2014 I had to write a web crawler as part of my job (don't ask) and to ensure I was a good internet citizen I created a PHP library
to parse robots.txt files so I could find out which parts of various websites I was allowed to crawl. The first version of the library
parsed the files into an intriguing tree data structure but that didn't really work very well at all so in 2016 it got rewritten to be
much more boring.

The nice thing about this project is it attracted a fair few external contributions and appears to still be in use.