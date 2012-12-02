---
layout: post
title: "Chewie 2.0 Beta"
date: 2012-12-01 22:42
comments: true
categories: [powershell, nuget]
---
NuGet isn't going away, so the best we can do is try to reduce the friction imposed upon us. [Chewie][] was a project created by Eric Ridgeway in June 2011 with the goal of bringing [Bundler][] like functionality to NuGet. While this solution has a number of shortcommings listed in my last [NuGet post][], I believe it is a much better solution that the vanilla NuGet experience. I rewrote Chewie based on many of Bundler's features, and have come to a point where I believe it is ready for a public preview. 

``` ps1 Sample .NugetFile
install_to 'lib'
chew Ninject [3.0.1.10]
chew Giles [0.1.4) dev -source "http://somethingrandom.feed.org"
chew NUnit

New-Group "Dev" {
  chew "Glimpse" "0.87"
}
```

``` ps1 Sample Commands
chewie init -path packages
chewie install Ninject
chewie install # installs all dependencies in the .NugetFile
chewie outdated # determines if any packages are outdated based on their version requirements
chewie update 
chewie uninstall
chewie convert
```

<iframe src="http://player.vimeo.com/video/54695717?badge=0" width="640" height="360" frameborder="0" webkitAllowFullScreen mozallowfullscreen allowFullScreen></iframe>


  [Chewie]: https://github.com/Ang3lFir3/Chewie
  [Bundler]: http://gembundler.com/
  [NuGet post]: /2012/10/nuget-youre-doing-it-wrong/