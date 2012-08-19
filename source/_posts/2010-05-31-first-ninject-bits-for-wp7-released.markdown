---
layout: post
title: "First Ninject bits for WP7 released"
date: 2010-05-31 21:34
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
I just released the first pre-release bits for WP7 support in Ninject. You can download them here. If you encounter any issues, please let me know. I have included signed and unsigned binaries in debug and release configurations.

Unit testing isn’t very easy for WP7. It was bizarre trying to debug the issues. The exact code that returns whether one type is assignable to another returns false in WP7. If you are trying to target WP7 and are familiar with Silverlight, you need to check out [this site](http://msdn.microsoft.com/en-us/library/ff426930\(VS.96\).aspx) to see what may hurt you; the [section on reflection](http://msdn.microsoft.com/en-us/library/ff426930\(VS.96\).aspx#Reflection) is rather painful. These are two of my favorites:

+ Open generics are not supported in Silverlight for Windows Phone.
+ The equality (==) operator for two equal MethodInfo objects does not return true in Silverlight for Windows Phone.

One thing to note is that this code is directly off the master branch for the 2.1 release. The 2.1 release has an enhanced resolution algorithm that may be a little different that what you previously expected – if you get an exception, you were most likely doing something ambiguous that Ninject wants straightened out. I will be blogging more about this later, but the binding resolution should now better handle what you are asking it to do.