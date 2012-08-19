---
layout: post
title: "Ninject.Extensions.Interception 2.0.1 Released"
date: 2010-03-19 17:39
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
I have just pushed new binaries and tagged the release on GitHub. There [was confusion](http://github.com/ninject/ninject.extensions.interception/issues/closed#issue/1) on how to deal with automatic module loading with the release that contained both proxy modules – every mailing list question pertained to this confusion, so it needed to be resolved. I have removed this release in favor of having the user download the extension with the dynamic proxy implementation of their choice.

There are two new projects: Ninject.Extensions.Interception.LinFu and Ninject.Extensions.Interception.DynamicProxy2 that contain the necessary classes for each dynamic proxy implementation.

This update also introduces LinFu.DynamicProxy for Silverlight 3.0 support.

You can download the latest release 2.0.1.0 from the GitHub [project downloads page](http://github.com/ninject/ninject.extensions.interception/downloads).