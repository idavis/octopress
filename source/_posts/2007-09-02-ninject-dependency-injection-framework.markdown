---
layout: post
title: "Ninject Dependency Injection Framework"
date: 2007-09-02 14:13
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
I have been playing with [Ninject](http://ninject.org/) for a couple days now and I am having a lot of fun. One feature I have been really digging is Contextual Binding. The kernel allows multiple modules to be loaded that define the bindings of your types. So depending on which modules and bindings are set, I can change the way other binding resolve. Here is an example:

Say I want to resolve a different type of car dependent on a selected engine. I have an interface IEngine and two classes V4, V6 that inherit from IEngine. If V4 is resolved by the kernel then the car returned would be a Civic; an Accord if a V6 is returned. Here is the code in custom modules:
``` csharp
internal class EngineModule : StandardModule
{
    public override void Load()
    {
        Bind<IEngine>().To( typeof ( V4 ) );
    }
}

internal class CarModule : StandardModule
{
    public override void Load()
    {
        IContext context = new StandardContext( Kernel, typeof ( IEngine ) );
        IBinding binding = Kernel.GetBinding<IEngine>( context );
        Type implementationType = binding.Provider.GetImplementationType( context, false );

        Bind<ICar>().To( typeof ( Accord ) ).OnlyIf( delegate { return implementationType == typeof ( V6 ) ) } );

        Bind<ICar>().To( typeof ( Civic ) ).OnlyIf( delegate { return implementationType == typeof ( V4 ) ) } );
    }
}
```
I can then load the modules into my kernel and use it to resolve cars:
``` csharp
internal class Program
{
    private static void Main( string[] args )
    {
        IModule[] modules = new IModule[] { new SettingsModule(), new EngineModule(), new CarModule() };
        IKernel kernel = new StandardKernel( modules );

        ICar car = kernel.Get<ICar>();
        Console.WriteLine( "I just got a brand new {0} {1}.", car.Manufacturer, car.Model );
        Console.WriteLine( "It has a {0} cylinder engine with {1} horsepower and {2} ft-lb torqe.", car.Engine.Cylinders, car.Engine.Horsepower, car.Engine.Torque );
        Console.WriteLine( "Time to start the Engine!" );
        car.Engine.Start();
        Console.WriteLine( "Sound good, but I am going to turn it off for now." );
        car.Engine.Stop();
        Console.ReadLine();
    }
}
```
Nate Kohari wrote a first cut [user guide](https://github.com/ninject/ninject/wiki) that is very helpful when getting started.