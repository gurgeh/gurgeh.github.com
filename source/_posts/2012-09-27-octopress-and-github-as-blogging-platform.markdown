---
layout: post
title: "Octopress and Github as a blogging platform"
date: 2012-09-27 16:37
comments: true
categories: 
published: true
---
I have switched from Blogspot to [Octopress](http://octopress.org). Any self respecting coder should realize that 1) blogging is text and 2) text should be in revision control. Also 3) blogging is public text, so the revision control can be on a public server, like Github.
<!--more-->

It was fairly easy, just follow the instructions on Octopress. If you are running Ubuntu, don't `apt-get rbenv`, use the latest version from Github instead. This is actually what the Octopress instructions tells you to do, but I ignored it and I am now happy to have survived.

The Octopress instructions for deploying to Github is also straightforward enough, as are the Github instructions for using your own domain name.

I used an [external script](https://gist.github.com/1765496) to migrate my Blogspot articles to my new site. It worked OK, but formatting was lost and the comments became static text. If you browse them, you will see that they currently look kinda ugly. When I have fixed them up a bit, I will begin forwarding all my old posts from Blogspot to fendrich.se.

There are not an abundance of themes yet, but you can always design one yourself. Picking colors and doing basic layout is easy. If you are design impaired, like I, there are a few ready-made [here](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes). Currently I use Darkstripes.

Also, a good blog has comments (and a good blog reader comments!), so I enabled [Disqus](http://disqus.com). 

Google Analytics support is built in, but just to be hip, I use [Clicky](http://getclicky.com) instead. It was simple to install. Just put the tracking code in `/source/_includes/custom/after_footer.html`.

I created my own favicon from my Twitter picture by using this [tool](http://www.favicon.cc/) and stuck it in the */source* directory. Possible because I am using Darkstripes I had to edit `/source/_includes/head.html` and change *favico.png* to *favico.ico*.

That's basically it. Hopefully this means I blog more. And better. And that the next post contains code. And perhaps even a pretty image. Or graph. And full sentences.