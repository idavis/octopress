---
layout: post
title: "Ninject available via Horn"
date: 2009-09-27 16:38
comments: true
categories: 
author: "Ian Davis"

# Github repositories
github_user: idavis
github_repo_count: 0
github_show_profile_link: true
github_skip_forks: true

# Twitter
twitter_user: ianfdavis
twitter_tweet_count: 4
twitter_show_replies: false
twitter_follow_button: true
twitter_show_follower_count: false
twitter_tweet_button: true
---
You can now build Ninject for both the 1.X and 2.0 code bases with [Horn](http://code.google.com/p/hornget) now. I would like to thank the [Horn developers and mailing list](http://groups.google.com/group/horn-development) for answering my questions and lending a hand.

If you haven’t heard of Horn, you should definitely check it out. [Taking Horn for a test drive](http://blog.bittercoder.com/PermaLink,guid,5614d98a-98f7-4343-b66c-3bf6da0707e0.aspx) is a great getting started article. To install Ninject using Horn is pretty easy:
```
// install Ninject 1.X
horn -install:ninject
 
// install Ninject 2.0
horn -install:ninject -version:2.0
```
You will notice that [Ninject 2.0](http://github.com/ninject/ninject) will compile a lot faster than 1.X as it does not have the [Castle](http://castleproject.org/) dependencies that the [Ninject 1.X](http://github.com/ninject/ninject1) code base has. Very cool stuff. I always liked [portage](http://www.gentoo.org/doc/en/handbook/handbook-x86.xml?part=2&chap=1) in Gentoo Linux; I am happy that it has a .NET sibling now.