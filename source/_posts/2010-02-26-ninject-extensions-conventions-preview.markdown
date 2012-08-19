---
layout: post
title: "Ninject.Extensions.Conventions Preview"
date: 2010-02-26 17:14
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
**EDIT**
The conventions extension has been completely rewritten with a new API for Ninject 3.0. This article references the 2.X releases.

I have been playing with the Ninject conventions binding extension syntax and usage model. It was originally based on the StructureMap assembly scanner, but I want to use the API in a different fashion.

I have been working on making the API more like LINQ in the way you think and use it. Here is a sampling of the new API that I am testing. Remember, you would never bind like this, ideally you would set up a few different scans to bind everything the way you want it.
``` csharp
IKernel kernel = new StandardKernel();
kernel.Scan(
    x =>
    {
        // set up the source collection of types to process
        x.FromCallingAssembly();
        x.From( from assembly in AppDomain.CurrentDomain.GetAssemblies()
                from type in assembly.GetExportedTypes()
                select type );
        x.FromAssemblyContaining<IGenericView>();
        x.FromAssembliesInPath( "." );
        x.FomAssembliesMatching( "*.Model.dll" );
        x.FromAssemblyContaining(new[] {typeof (IGenericView), typeof (StringView)});
 
        // Filter out what you want
        x.Where( type => !type.IsAbstract );
        x.WhereTypeIsInNamespace( "Base.Model" );
        x.WhereTypeIsInNamespaceOf<StringView>();
        x.WhereTypeIsNotInNamespace( "Base.Test" );
        x.WhereTypeIsNotInNamespaceOf<StandardKernel>();
        x.WhereTypeInheritsFrom<IGenericView>();
 
        // Exclude items even if they match your Where criteria
        x.Excluding<INinjectSettings>();
        x.Excluding( new[] {typeof (IServiceProvider), typeof (IInitializable)} );
 
        // Include items whether or not they match your Where criteria
        x.Including<DefaultView>();
 
        // Tell the scanner how to determine bindings
        x.BindWithDefaultConventions();
 
        // Declare the binding scope for scanned members
        x.InTransientScope();
    } );
```
Some API calls will not be available on all platforms due to security restrictions of course.