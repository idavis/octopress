---
layout: post
title: "Using Ninject with TopShelf and Quartz"
date: 2010-05-22 21:30
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
{% blockquote %}
Disclaimer: I am neither an expert in TopShelf nor Quartz, this is just me trying to plug them together with Ninject.
{% endblockquote %}
The fist thing we need to have in order to use Quartz is to create a job that will be executed. We will call ours the VerifyEmailServerJob.
``` csharp
public class VerifyEmailServerJob : IJob
{
    #region Implementation of IJob
 
    public void Execute( JobExecutionContext context )
    {
        // TODO: Verify the email server is working correctly...
    }
 
    #endregion
}

```
Now that we have a job to run, we will create a windows service that will create the job details and schedule the job. The windows service essentially acts as a host for the jobs.
``` csharp
public interface IWindowsService
{
    void Start();
    void Stop();
}
 
public partial class MonitoringService : ServiceBase, IWindowsService
{
    private readonly JobManager _jobManager;
    private readonly List<IScheduler> _schedulers = new List<IScheduler>();
 
    public MonitoringService( JobManager jobManager )
    {
        _jobManager = jobManager;
        InitializeComponent();
    }
 
    public void Start()
    {
        // TODO: Pull from config file.
        var trigger = new CronTrigger( "name", "group", "0 0 4/8 * * MON-FRI" );
        IScheduler scheduler = _jobManager.Add<VerifyEmailServerJob>( trigger );
        _schedulers.Add( scheduler );
    }
 
    protected override void OnStop()
    {
        _schedulers.ForEach( scheduler => scheduler.Shutdown( false ) );
        _schedulers.Clear();
    }
}
```
In order to inject our services into the jobs, we need to take control of the job activation pipeline. Quartz allows us to do this by using a implementation of IJobFactory. In my example I am injecting a creation callback function that will use the Ninject kernel to activate the job and add any dependencies without the IJobFactory knowing of the kernel. I am also throwing in my own JobManager to cut down on the amount of code I need to write to add a job. But note that we have to tell the scheduler which JobFactory to use when activating.
``` csharp
public class JobFactory : IJobFactory
{
    private readonly Func<Type, IJob> _creationCallback;
 
    public JobFactory( Func<Type, IJob> creationCallback )
    {
        _creationCallback = creationCallback;
    }
 
    #region Implementation of IJobFactory
 
    public IJob NewJob( TriggerFiredBundle bundle )
    {
        JobDetail jobDetail = bundle.JobDetail;
        Type jobType = jobDetail.JobType;
        var job = _creationCallback( jobType );
        return job;
    }
 
    #endregion
}
 
public class JobManager
{
    private readonly IJobFactory _jobFactory;
    private readonly ISchedulerFactory _schedulerFactory;
 
    public JobManager( ISchedulerFactory schedulerFactory, IJobFactory jobFactory )
    {
        _schedulerFactory = schedulerFactory;
        _jobFactory = jobFactory;
    }
 
    public IScheduler Add<T>( Trigger trigger ) where T : IJob
    {
        IScheduler scheduler = _schedulerFactory.GetScheduler();
        scheduler.JobFactory = _jobFactory;
        scheduler.Start();
        var jobDetail = new JobDetail( typeof (T).Name, typeof (T) );
        scheduler.ScheduleJob( jobDetail, trigger );
        return scheduler;
    }
}
```
Now with out jobs, triggers, and service foundation, all we need to do is configure TopShelf to run our service and configure our kernel to inject our dependencies.
``` csharp
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
                        x.SetDisplayName( "Monitoring Service" );
                        x.SetServiceName( "Monitor" );
                        x.SetDescription( "Monitoring Service" );
                        x.ConfigureService<MonitoringService>( kernel );
                        x.RunAsLocalSystem();
                    } );
 
            Runner.Host( cfg, args );
        }
    }
 
    private static IKernel CreateKernel()
    {
        var kernel = new StandardKernel();
        kernel.Bind<IJobFactory>().To<JobFactory>();
        kernel.Bind<Func<Type, IJob>>().ToConstant( type => (IJob) kernel.Get( type ) );
        kernel.Bind<ISchedulerFactory>().To<StdSchedulerFactory>();
        kernel.Bind<JobManager>().ToSelf().InSingletonScope();
        kernel.Bind<IWindowsService>().To<MonitoringService>();
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
```
That about does it! We have a service configured and controlled by TopShelf, running scheduled jobs using Quartz, and dependencies and jobs created by Ninject.