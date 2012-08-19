---
layout: post
title: "Open Source Projects"
date: 2008-11-28 15:12
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
I had two open source projects: Ensurance, and IMAPI.Net.

+ Ensurance did not really fit into any particular group. It was a tool for me while Spec# was still a research project at MS. I wanted to be able to use pre/post conditions, parameter constraints, and tie them into logging and debugging. I don’t think that Spec# will do the last bit, but I am really looking forward to its release.
+ IMAPI.Net is a C# cd authoring library that wraps the IMAPI system in windows to allow programmers of any application to write to CD. It supports data and audio discs. It was expensive to test the library and the COM issues were a huge problem for a while.
+ + PInvoke.net was a good help for the most part, but the marshalling took a long time to get right. I started the project back in .net 1.1 while working on my BS – and it worked. With each .NET release the project would have been easier to write with the COM interop support increasing. I used to be the lead of the XPBurn component project on GotDotNet before it shut down; in truth though, I had stopped supporting it a couple months before that. No one would read directions, read the forum, follow any kind of guidelines and would essentially expect me to figure out why they are using the code wrong. It became too much hassle in the end for what I got out of the project. It was a great learning experience though.
Both projects are essentially dead and I will be closing down the Ensurance project on CodePlex and hosting the code for reference on GitHub. I will leave IMAPI.Net up on <del>google code</del> GitHub in case anyone ever finds use for the code. If I get the time I will try to add my PInvoke code to the wiki so others don’t have to work as hard as I did.