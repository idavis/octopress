---
layout: post
title: "TFS Shell Extension"
date: 2010-01-29 17:05
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
If you are using the TFS Shell Extension and you get an error dialog when choosing Team Foundation Server -> Reconnect

```
—————————
Unable to Reconnect
—————————
Unable to connect to Team Foundation Server
—————————
OK
—————————
```

There is a workaround. TFS extension only supports windows credentials at the moment, so if the server is using domain authentication, you can try to open your TFS server url in an IE window, enter your domain user credentials and password. Now try to reconnect and you should be able to use the shell extension.

If you are using Windows 7, maybe Vista as well, you can use the Credential Manager in the Control Panel to manage the caching of this information without IE.