---
layout: post
title: "C++ if statement oddity"
date: 2008-11-28 15:16
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
I found that you can do work such as assigning a variable and executing a method outside of the main expression of an if statement. For example:
``` c++
if( hRes = GetApplicationState(), NT_SUCCESS(hRes) ) {
    // Do something
}
```