---
layout: post
title: "Ninject.Extensions.Interception 2.0 Released!"
date: 2010-03-06 17:16
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
On the heels of the [Ninject 2.0 release](/2010/02/ninject-forever/), I am happy to announce the release of the [Ninject Interception Extension 2.0](http://github.com/ninject/ninject.extensions.interception). This release provides API backward compatibility for the 1.5 version as well as new method interception, enhanced binding syntax, automatic property changed notification, as well as support for Castle DynamicProxy 2.2 and LinFu DynamicProxy. I will be writing a follow up post about the new features and usage patterns.

I want to thank [Krzysztof Koźmic](http://kozmic.pl/) for his help with the Castle DynamicProxy 2.2 integration and [Michael Hart](http://stackoverflow.com/users/58173/michael-hart) for letting me merge his Ninject 1.5 method interception code into the new extension.

You will find downloads on the GitHub dowload page for .NET 3.5, Silverlight 3, and Mono 2.0. For those using automatic module loading or only want to use one DynamicProxy dependency without worrying about delay loading, there are three .NET 3.5 downloads: Castle+LinFu, Castle, and LinFu. Go get it!