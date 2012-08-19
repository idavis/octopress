---
layout: post
title: "WinDgb script when debugging drivers"
date: 2010-05-01 18:59
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
If you ever happen to be writing a device driver and you want to have WinDbg configured to run a set of scripts, this configuration for WINDBG.INI may be of help. If it doesn’t mean anything to you, don’t worry, nothing to see here other than pain. Replace anything in {} with your own paths and file names. This is configured for kernel debugging over 1394.

```
[WinDbg Location]
C:Program FilesDebugging Tools for Windows (x86)windbg.exe
[WinDbg CmdLines]
-T "Initial WinDbg Line" -c "SQE;BL"
-T "Default" -Q -QY -y http://msdl.microsoft.com/download/symbols;C:{PathToBinFiles} -k 1394:channel=44,symlink=Channel -c "SQE;BL;.kdfiles -m Systemrootsystem32DRIVERS{YourDriver1}.sys C:{PathToBinFiles}{YourDriver1}.sys;.kdfiles -m Systemrootsystem32DRIVERS{YourDriver2}.sys C:{PathToBinFiles}{YourDriver2}.sys;.kdfiles"
```