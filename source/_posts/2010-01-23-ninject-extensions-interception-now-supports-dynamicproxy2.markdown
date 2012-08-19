---
layout: post
title: "Ninject.Extensions.Interception now supports DynamicProxy2"
date: 2010-01-23 16:45
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
Seeing that people wanted to use Castle DynamicProxy2 with Ninject for interception, I ported the support from Ninject 1.5 and we now have support for using DynamicProxy2 built-in for the Ninject 2.0 extension.

DynamicProxy2:
``` csharp
IKernel kernel = new StandardKernel(new DynamicProxy2Module());
```
LinFu:
``` csharp
IKernel kernel = new StandardKernel(new LinFuModule());
```
**Note: Breaking Change**

InterceptionModule is now abstract. The proxy specific modules need to be used now. This base module is used to wire up the interception pipeline leaving inheritors to only registry proxy factories.