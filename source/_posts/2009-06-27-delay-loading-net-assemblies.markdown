---
layout: post
title: "Delay loading .Net assemblies"
date: 2009-06-27 15:35
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
I had to figure out how to make an application run as its native type while loading only the proper dlls for that platform – when the application runs on x64, load only x64 assemblies so that the application will run fully as x64 and the same for x86. The answer is counterintuitive and not straightforward.

Below is an example application that exhibits this behavior. The Amd64Library.Foo is defined in an assembly compiled for x64; the Win32Library.Foo is similarly defined in an x86 assemly. The Loader application is compiled as ‘Any CPU’ so it will load natively on either platform as long as we load the correct dependency assemblies. Remember that the using statement, is just a compiler hint and doesn’t effect the created MSIL application.
``` csharp
#region Using Directives
 
using System;
using Win32Library;
 
#endregion
 
namespace LoaderApplication
{
    internal class Program
    {
        private static void Main( string[] args )
        {
            //RunLoadingBothAssemblies(); // will crash.
            RunLoadingOneAssembly();
            Console.ReadLine();
        }
 
        private static void RunLoadingBothAssemblies()
        {
            if ( IntPtr.Size == 4 )
            {
                var foo = new Foo();
                foo.Bar();
            }
            else
            {
                var foo = new Amd64Library.Foo();
                foo.Bar();
            }
        }
 
        private static void RunLoadingOneAssembly()
        {
            if ( IntPtr.Size == 4 )
            {
                LoadWin32Foo();
            }
            else
            {
                LoadAmd64Foo();
            }
        }
 
        private static void LoadAmd64Foo()
        {
            var foo = new Amd64Library.Foo();
            foo.Bar();
        }
 
        private static void LoadWin32Foo()
        {
            var foo = new Foo();
            foo.Bar();
        }
    }
}
```

The LoaderApplication references both x64 and x86 assemblies – yes, at the same time. The key is that the assemblies aren’t loaded into the process until the last possible moment. If you look at the RunLoadingBothAssemblies method, the moment you enter the method, the .NET runtime will load the assemblies for the types in that method. This means it will try to load the x64 library into a 32bit process and vise versa resulting in errors. The RunLoadingOneAssembly determines which platform is being used and then makes another call so that only the library needed is loaded.

There is a caveat to this method. The assemblies need to use separate namespaces or different type names. If both assemblies use the same namespace, you will run into type resolution issues as the compiler won’t know which class you are trying to load while referencing both assemblies. If you have source access to the assembly that has to be compiled for each platform, #if statements will allow you to easily adjust the namespaces or type names.

You can watch this application run on x64 and the process name will not have *32 as it will be running natively. So, if you want to have a single build of your application, but run on x64 and x86, this is one way to go.