---
layout: post
title: "Windows Services with Ninject and TopShelf"
date: 2010-05-18 21:22
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
I was testing out TopShelf and here is an example I put together in order to have Ninject activate the services and their dependencies:
``` csharp
#region Using Directives
 
using System;
using System.ServiceProcess;
using YourCorp.Services;
using Ninject;
using Topshelf;
using Topshelf.Configuration;
using Topshelf.Configuration.Dsl;
 
#endregion
 
namespace YourCorp
{
    internal static class Program
    {
        private static void Main( string[] args )
        {
            using ( IKernel kernel = CreateKernel() )
            {
                RunConfiguration cfg =
                    RunnerConfigurator.New(
                        x =>
                        {
                            x.SetDisplayName( "Your Service" );
                            x.SetServiceName( "YourService" );
                            x.SetDescription( "Your Service" );
                            x.ConfigureService<MonitoringService>( kernel );
                            x.ConfigureService<ReportingService>( kernel );
                            x.RunAsLocalSystem();
                        } );
 
                Runner.Host( cfg, args );
            }
        }
 
        private static IKernel CreateKernel()
        {
            var kernel = new StandardKernel();
            kernel.Bind<IWindowsService>().To<MonitoringService>();
            kernel.Bind<IWindowsService>().To<ReportingService>();
            return kernel;
        }
    }
 
    public static class SyntaxExtensions
    {
        public static void ConfigureService<T>( this IRunnerConfigurator runnerConfigurator, IKernel kernel )
            where T : ServiceBase, IWindowsService
        {
            runnerConfigurator.ConfigureService<T>(
                c =>
                {
                    c.HowToBuildService( serviceName => kernel.Get<T>() );
                    c.Named( typeof (T).Name );
                    c.WhenStopped( s => s.Stop() );
                    c.WhenStarted( s => s.Start() );
                } );
        }
    }
 
    public interface IWindowsService
    {
        void Start();
        void Stop();
    }
}
```