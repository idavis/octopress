---
layout: post
title: "Continuous Build Systems"
date: 2008-12-15 15:22
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
I have gone through so many tools trying to create the right build environment that it is a bit disheartening. I have set up multiple continuous integration systems and it keeps getting easier, but there is still a bit too much friction.

I tried to look into using Team Foundation Server and Team Build, but the cost is very high for a startup (even with the top msdn subscription). This is compounded when you have external developers that must also hook into the system.

I have fallen for TeamCity but with one deal breaker for me: no FinalBuilder support. I submitted some feedback and I received a detailed response from JetBrains; I have always received great customer support from JetBrains:

{% blockquote %}
There are several ways to support FinalBuilder: 
1. You may start using CommandLine runner to start the process. 
2. To Add custom FinalBuilder reporting you may use TeamCity service messages. 
   Please refer to 
http://www.jetbrains.net/confluence/display/TCD4/Build+Script+Interaction+with+TeamCity 
   documentation article on service messages for details 
3. You may start FinalBuilder from any other build scripts, 
   for example NAnt, MSBuild, Ant, Maven. 
   If  you  need  custom  logging,  you  may  consider using TeamCity 
   Service messages as well 
4. Write a build runner plugin for TeamCity. There are two 
   public available examples on the build runners at: 
http://www.jetbrains.net/confluence/display/TW/Rake+Runner 
   and 
http://svn.jetbrains.org/teamcity/plugins/fxcop/ 
   Please feel free asking any questions on the integration. 
5. Post an issue to our tracker at 
   http://www.jetbrains.net/tracker
{% endblockquote %}

I checked out the code for the build runner plugins, but it is too much work. It is hard not to sound lazy in saying that, but I haven’t used java since my sophomore  year in college, and it doesn’t sit high on my list to spend hours getting a build runner going (I am looking for less friction, not more). I chose the 5th option and [submitted a feature request TW-6442](http://www.jetbrains.net/tracker/issue/TW-6442). With FinalBuilder support I don’t need any other build runners/tools as FinalBuilder most likely wraps them up in its UI.

One other feature that I would like to have, but am hesitant to request right now, is to be able to selected a successful build, and run a publish script (through the web interface (see Cruise)) that can tag the build and use the build  artifacts to deploy that build to Dev/Stage/Production environment.