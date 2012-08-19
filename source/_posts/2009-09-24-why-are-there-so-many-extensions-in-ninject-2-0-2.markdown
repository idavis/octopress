---
layout: post
title: "Why are there so many extensions in Ninject 2.0?"
date: 2009-09-26 16:25
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
We really wanted to [Ninject 2.0](http://github.com/ninject/ninject) as small as possible. Ninject 1.X was a huge (to us); a monolithic kernel that could do anything. While it was nice to be able to do anything, it came at the cost of file size, memory footprint, complexity, and more. It also presented challenges for extension authors as there was a considerable amount of code and process to understand.

In Ninject 2.0, we have tried to keep the Ninject.dll under 90Kb and provide a number of extension points. We also strove for usability, compatibility, and extensibility. To do this, we had to break support for .NET 2.0; however, we included .NET 3.5, Silverlight 2.0 & 3.0, Mono 2.0, and .NET Compact Framework 3.5 support.

One thing you may notice is that a couple of extensions used to be part of the Ninject kernel. These features were extracted as many people don’t need them. For example, logging and interception are no longer available by default, but including them is as simple as dropping the dlls into the same directory as Ninject and the extensions will be loaded automatically.

For authors of [Ninject 1.X](http://github.com/ninject/ninject1) extensions, the upgrade for 2.0 is pretty easy. Having done it many times, I feel comfortable in this assessment. There is only one gotcha – when you are unit testing your extension; make sure to disable extension auto-loading. This is done by passing a NinjectSettings object into the kernel .ctor with ninjectSettings.LoadExtensions = false. Without this, the Ninject kernel will always load you extension, even when your test doesn’t need it.

We believe that this highly extensible, microkernel approach will serve us, the Ninject developers, and the user community very well. With the plethora of extensions in Ninject 2.0, you will essentially be able to create designer kernels to fit your exact needs, without bloat.