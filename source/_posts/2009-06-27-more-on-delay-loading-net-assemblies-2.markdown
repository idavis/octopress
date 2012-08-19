---
layout: post
title: "More on delay loading .Net assemblies"
date: 2009-06-27 15:38
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
You can build off of the [previous article](/2009/06/delay-loading-net-assemblies/) and make dealing with the dual libraries much easier using an IoC container. Instead of building an interface that constantly has to check which code path to execute based on the current platform, you can load the services into an IoC container. Once you know what platform you are on, you can reference the type forcing its assembly to be loaded into your current AppDomain (as shown previously). Once done, you can get the assembly through a few different ways and build up services with an IoC container.

You can use the assembly scanner in StructureMap, or other mechanisms in another container, to go through the assembly and build up services that you can run. If your platform specific assemblies reference a third contract definition assembly then you are in great shape. If not, you might use string resolution (container.Get(“MyService”)) to obtain the object and either duck type or reflection invoke its members.