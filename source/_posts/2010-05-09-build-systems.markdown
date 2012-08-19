---
layout: post
title: "Build Systems"
date: 2010-05-09 19:13
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
```
$:>call begin-rant
```
{% blockquote %}
I am impeded and embittered by the current state of builds systems. I need to get ready to make Windows Phone 7 and Silverlight 4 builds, but the tooling doesn’t have support for them. On a side note, why the heck did I have to install VS2010 Express in order to get the WP7 SDK? #MSFAIL! It is hard to justify staying with builds systems based in XML that are manually edited and impossible to debug. In the short term, I am probably going to create my WP7 and SL4 builds by hand – I do not say this lightly, I feel sick doing so, but the level of effort to get something working properly is too high for the amount of time I have at the moment.
{% endblockquote %}
```
Disclaimer, below is still a rant, but less so than above.
```
The state of build systems is incredibly sad at the moment. Open source projects have no choice but to use some open source build tool adding friction to the development of the project – not because they are open source, but rather because they are poor tools. What choices do we really have – XML, Ruby, PowerShell, and a UI wrapping one of them; you can name others, but there is no traction on them so far as I can tell. Along with each is a constant ridicule by a particular group touting the magnificence of their own choice in tooling. The UI tools have the added hatred in that they are not free and the disdain of point and click.

Just to make it clear, I am biased, I love FinalBuilder – a non free UI build tool; but I can’t use it for open source and the extension model is a pain. I do not want to learn yet another language in order to write my builds scripts, and I absolutely do not want to write XML. This eliminates all of my needs, so I have to sacrifice one of my requirements in order to get the job done.

Below is a list of the major tools available as I currently see them. I know there are many others, but they aren’t important to me.

#### Language based
+ psake (PowerShell Build Automation Tool)
+ Rake (Ruby Make)
+ Fake (F# Make)
+ BooBs (Boo Build System)

#### XML Based
+ NAnt  (.NET Build Tool)
+ MSBuild

#### Visual Editor
+ FinalBuilder
+ VisualBuild

PowerShell, I wanted to love you, but found you just frustrating to use. I love that you can access .NET libraries, but your debugging and editing sucks. In discussions I have had with others, Ruby’s niche seems to be rails and DSLs. With PowerShell and Ruby, you have to have their runtime on your machine. Many developers I know won’t install Ruby on their computers, and I really don’t want to force users of my open source software to install PowerShell or Ruby – it feels like a significant failure on my part were I to do so. On the other hand, do users really need to be making builds? Can they just use VS? MSBuild and NAnt, you are dead to me. The nightmares of past attempts to ‘debug’ XML haunt me.

I am still in a lose-lose situation. I realize that I am trying to deal with two separate issues that seem to be the root of the problem. First, trying to stay in a .NET environment and not requiring yet another external dependency. Second, the poor tooling. I have been very tempted to write an open source version of FinalBuilder, but most .NET developers I have mentioned it to have asked “What is FinalBuilder?” – which doesn’t inspire me to invest my time.

Right now, I see two front runners that will emerge: Visual Tools (FinalBuilder, VisualBuild), and Ruby (IronRuby). I think that the aversion to Ruby will push .NET developers to use IronRuby specifically (but this is still an extra install  ). If I am going to use something other than FinalBuilder, I want it to be a real part of my product. I don’t want a second class build tool. I want something that goes under the same scrutiny as my production code. I want to make my scripts a full part of my work: to refactor, test, debug, review, and version control my build scripts.

Right now, I think I need to learn Ruby, in-depth, and see what can be done. The Albacore rake tasks seem to be a good start, so I will have to see where it leads to. One interesting thing I picked up when evaluating current systems and looking to build my own is that they are really all just consumers of a common data model. If you could create a detailed model of the options available for a particular task, you should theoretically be able to generate psake, rake, fake, build files from the data model.