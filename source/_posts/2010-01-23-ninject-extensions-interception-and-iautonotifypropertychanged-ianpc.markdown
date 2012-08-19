---
layout: post
title: "Ninject.Extensions.Interception and IAutoNotifyPropertyChanged (IANPC)"
date: 2010-01-23 16:49
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
After having conversation on twitter with [Jonas Folleso](http://twitter.com/follesoe) about an automatic INotifyPropertyChanged implementation, I couldn’t resist digging in and seeing what I can do. He also has [a blog post](http://jonas.follesoe.no/AutomaticINPCUsingDynamicProxyAndNinject.aspx) that is definitely worth reading on this subject. He solved the same problem through the [IProvider](http://github.com/ninject/ninject/blob/master/src/Ninject/Activation/IProvider.cs) functionality in [Ninject](http://ninject.org/). The code is based on his initial implementation. I had originally planned to do this as a demo of how to extend [Ninject.Extensions.Interception](http://github.com/ninject/ninject.extensions.interception), but I decided to roll it into the trunk of the project instead. There is a set of goals I wanted to accomplish with the IANPC implementation

+ Create a property [[NotifyOfChanges]](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/Attributes/NotifyOfChangesAttribute.cs) that would indicate that a particular property should be intercepted and participate in IANPC
+ Be able to use that same property on a class to indicate that you would like to proxy all available properties for IANPC
+ Have additional dependency property notifications so multiple events will be triggered.
+ Create a property [[DoNotNotifyOfChanges]](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/Attributes/DoNotNotifyOfChangesAttribute.cs) that would indicate that a particular property should not notify property changes, but will still be available for other interception schemes. This is only applicable if [NotifyOfChanges] was applied at the class level.

We first need to start with a contract that the ViewModel will need to adhere to in order to participate in our party. We could add more options to suppress messages dynamically, but let’s keep it clean for now.

``` csharp
public interface IAutoNotifyPropertyChanged : INotifyPropertyChanged
{
    void OnPropertyChanged( string propertyName );
}
```
Given this, we now have a callback for when we detect that a property has changed. Now we need a sample view model Implementation that will flex all of our interception goals:
``` csharp
public class ViewModelBase : IAutoNotifyPropertyChanged
{
    #region IAutoNotifyPropertyChanged Members
 
    public event PropertyChangedEventHandler PropertyChanged;
 
    public void OnPropertyChanged( string propertyName )
    {
        PropertyChangedEventHandler handler = PropertyChanged;
        if ( handler != null )
        {
            handler( this, new PropertyChangedEventArgs( propertyName ) );
        }
    }
 
    #endregion
}
 
public class ViewModel : ViewModelBase
{
    [NotifyOfChanges( "City", "State" )]
    public virtual int ZipCode { get; set; }
 
    public virtual string City
    {
        get { return string.Empty; }
    }
 
    public virtual string Address { get; set; }
 
    [DoNotNotifyOfChanges]
    public virtual string DoNotNotifyChanges { get; set; }
}
 
[NotifyOfChanges]
public class ViewModelWithClassNotify : ViewModelBase
{
    [NotifyOfChanges( "City", "State" )]
    public virtual int ZipCode { get; set; }
 
    public virtual string City
    {
        get { return string.Empty; }
    }
 
    public virtual string Address { get; set; }
 
    [DoNotNotifyOfChanges]
    public virtual string DoNotNotifyChanges { get; set; }
}
```
Now that we have our contract and attributes we need to integrate them into the planning pipeline for the interception extension. We want to hook into the planning pipeline by implementing IPlanningStrategy::Execute(IPlan). When Ninject resolves a service it builds an activation plan. Prior to creating the instance, Ninject loops through all of the registered planning strategies executing the plan, configuring directives, and getting things ready for the activation strategies. Here we will find candidate properties and register the interceptors associated with them.
``` csharp
public class AutoNotifyInterceptorRegistrationStrategy : InterceptorRegistrationStrategy
{
    public AutoNotifyInterceptorRegistrationStrategy( IAdviceFactory adviceFactory, IAdviceRegistry adviceRegistry )
        : base( adviceFactory, adviceRegistry )
    {
    }
 
    public override void Execute( IPlan plan )
    {
        if ( !typeof (IAutoNotifyPropertyChanged).IsAssignableFrom( plan.Type ) )
        {
            return;
        }
 
        IEnumerable<MethodInfo> candidates = GetCandidateMethods( plan.Type );
 
        RegisterClassInterceptors( plan.Type, plan, candidates );
 
        foreach ( MethodInfo method in candidates )
        {
            PropertyInfo property =
                 method.GetPropertyFromMethod( method.DeclaringType );
            NotifyOfChangesAttribute[] attributes =
                 property.GetAllAttributes<NotifyOfChangesAttribute>();
 
            if ( attributes.Length == 0 )
            {
                continue;
            }
 
            RegisterMethodInterceptors( plan.Type, method, attributes );
 
            // Indicate that instances of the type should be proxied.
            if ( !plan.Has<ProxyDirective>() )
            {
                plan.Add( new ProxyDirective() );
            }
        }
    }
 
    protected override void RegisterClassInterceptors( Type type, IPlan plan, IEnumerable<MethodInfo> candidates )
    {
        NotifyOfChangesAttribute[] attributes =
             type.GetAllAttributes<NotifyOfChangesAttribute>();
 
        if ( attributes.Length == 0 )
        {
            return;
        }
 
        foreach ( MethodInfo method in candidates )
        {
            PropertyInfo property = method.GetPropertyFromMethod( method.DeclaringType );
            if ( !property.HasAttribute<DoNotNotifyOfChangesAttribute>() )
            {
                RegisterMethodInterceptors( type, method, attributes );
            }
        }
 
        // Indicate that instances of the type should be proxied.
        if ( !plan.Has<ProxyDirective>() )
        {
            plan.Add( new ProxyDirective() );
        }
    }
 
    protected override bool ShouldIntercept( MethodInfo methodInfo )
    {
        if ( !IsPropertySetter( methodInfo ) )
        {
            return false;
        }
 
        if ( IsDecoratedWithDoNotNotifyChangesAttribute( methodInfo ) )
        {
            return false;
        }
        return base.ShouldIntercept( methodInfo );
    }
 
    private static bool IsPropertySetter( MethodBase methodInfo )
    {
        return methodInfo.IsSpecialName && methodInfo.Name.StartsWith( "set_" );
    }
 
    private static bool IsDecoratedWithDoNotNotifyChangesAttribute( MethodInfo methodInfo )
    {
        PropertyInfo propertyInfo =
            methodInfo.GetPropertyFromMethod( methodInfo.DeclaringType );
        return
            ( propertyInfo != null &&
              propertyInfo.GetOneAttribute<DoNotNotifyOfChangesAttribute>() != null );
    }
}
```
That wasn’t too bad, right? We looked at the plan and registered interceptors when applicable that will be attached during the instance activation. Now we need to create the attributes for the planning strategy to process.

We need to create a simple attribute for [[DoNotNotifyOfChanges]](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/Attributes/DoNotNotifyOfChangesAttribute.cs) – we could have just used [[DoNotIntercept]](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/Attributes/DoNotInterceptAttribute.cs), but I still want to allow other interception schemes to be able to proxy these properties. Even though [DoNotIntercept] is its base class, we do not look at inheritance of attributes when processing the activation plans.
``` csharp
[AttributeUsage( AttributeTargets.Property, AllowMultiple = false, Inherited = true )]
public class DoNotNotifyOfChangesAttribute : DoNotInterceptAttribute
{
}
```
The NotifyOfChangesAttribute is a little more complex, but still pretty easy. We want to inherit from InterceptAttribute, but we need to change the targets to classes and properties only. The base attribute was able to attach to methods which doesn’t make sense for us. We use the proxy request to construct an interceptor based on the target and notification settings. The tricky thing here is that the default interceptor we are using is an open generic, so we have to close the type before creating the interceptor instance.
``` csharp
[AttributeUsage( AttributeTargets.Class | AttributeTargets.Property, AllowMultiple = false, Inherited = true )]
public class NotifyOfChangesAttribute : InterceptAttribute
{
    private static readonly Type InterceptorType = typeof (AutoNotifyPropertyChangedInterceptor<>);
 
    public NotifyOfChangesAttribute( params string[] notifyChangeFor )
    {
        NotifyChangeFor = notifyChangeFor;
    }
 
    public string[] NotifyChangeFor { get; private set; }
 
    public override IInterceptor CreateInterceptor( IProxyRequest request )
    {
        Type targetType = request.Target.GetType();
        Type closedInterceptorType = InterceptorType.MakeGenericType( targetType );
        var interceptor = (IInterceptor) request.Context.Kernel.Get( closedInterceptorType );
        return interceptor;
    }
}
```
The interceptor for the first implementation is pretty simple. We pass the call along to invoke the original setter, then trigger the event from the proxy instance and any dependent properties.
``` csharp
public class AutoNotifyPropertyChangedInterceptor<TViewModel>
    : IInterceptor where TViewModel : IAutoNotifyPropertyChanged
{
    #region IInterceptor Members
 
    public void Intercept( IInvocation invocation )
    {
        invocation.Proceed();
 
        MethodInfo methodInfo = invocation.Request.Method;
        var model = (TViewModel) invocation.Request.Proxy;
        model.OnPropertyChanged( methodInfo.GetPropertyFromMethod( methodInfo.DeclaringType ).Name );
 
        ChangeNotificationForDependentProperties( methodInfo, model );
    }
 
    #endregion
 
    private static void ChangeNotificationForDependentProperties( MethodInfo methodInfo,
                                                                  IAutoNotifyPropertyChanged model )
    {
        if ( NoAdditionalProperties( methodInfo ) )
        {
            return;
        }
 
        string[] properties = GetAdditionalPropertiesToNotifyOfChanges( methodInfo );
 
        foreach ( string propertyName in properties )
        {
            model.OnPropertyChanged( propertyName );
        }
    }
 
    private static bool NoAdditionalProperties( MethodInfo methodInfo )
    {
        PropertyInfo propertyInfo = methodInfo.GetPropertyFromMethod( methodInfo.DeclaringType );
        return ( propertyInfo == null || propertyInfo.GetOneAttribute<NotifyOfChangesAttribute>() == null );
    }
 
    private static string[] GetAdditionalPropertiesToNotifyOfChanges( MethodInfo methodInfo )
    {
        PropertyInfo propertyInfo = methodInfo.GetPropertyFromMethod( methodInfo.DeclaringType );
        var attribute = propertyInfo.GetOneAttribute<NotifyOfChangesAttribute>();
        return attribute.NotifyChangeFor;
    }
}
```
In order to attach the AutoNotifyInterceptorRegistrationStrategy to the planning pipeline, we simply add it to the kernel components associated with IPlanningStrategy during the InterceptionModule’s Bind() method.
``` csharp
...
Kernel.Components.Add<IPlanningStrategy, AutoNotifyInterceptorRegistrationStrategy>();
...
```
That’s it! We now have natively integrated, automatic INotifyPropertyChanged proxy generation. We can do more work to actually compare the existing and new values for changes, but we can enhance the interface later or move to support more flexible interceptor generation for IANPC.
