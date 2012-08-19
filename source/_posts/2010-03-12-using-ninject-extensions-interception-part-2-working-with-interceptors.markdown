---
layout: post
title: "Using Ninject.Extensions.Interception Part 2 : Working with Interceptors"
date: 2010-03-12 17:26
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
### Overview

In [Part 1](/2010/03/using-ninject-extensions-interception-part-1-the-basics) we covered the basics of the extension and introduced the IInterceptor interface. Now we can move more in depth and look at the interceptor system composed of

+ Custom Interceptors
+ Interception via Attributes

With these, we can define a rich set of interception capabilities in our applications.

### Custom Interceptors

The base of the binding interception is the very simple IInterceptor interface:
``` csharp
public interface IInterceptor
{
    /// <summary>
    /// Intercepts the specified invocation.
    /// </summary>
    /// <param name="invocation">The invocation to intercept.</param>
    void Intercept( IInvocation invocation );
}
```
The idea behind and IInterceptor is to take the place of some kind of action. This is very simple and makes creating an custom interceptor very easy. In order to do it properly, however, you must use the IInvocation object you are given. Below is the most basic interceptor that you can create which resumes execution of the real method.
``` csharp
public class MinimalInterceptor : IInterceptor
{
    public void Intercept( IInvocation invocation )
    {
        invocation.Proceed();
    }
}
```
The invocation.Proceed() call is crucial! Since a call can have multiple interceptors, we need a way to hand off execution to the next downstream interceptor. The Proceed() call will either invoke the next interceptor in the chain, or invoke the target intercepted method if their are no more interceptors.

The extension currently ships with three interceptors

+ [SimpleInterceptor](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/SimpleInterceptor.cs)
+ [ActionInterceptor](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/ActionInterceptor.cs)
+ [AutoNotifyPropertyChangedInterceptor](https://github.com/ninject/ninject.extensions.interception/blob/master/src/Ninject.Extensions.Interception/AutoNotifyPropertyChangedInterceptor.cs) (covered in [IANPC post](/2010/01/ninject-extensions-interception-and-iautonotifypropertychanged-ianpc/))

#### SimpleInterceptor
The SimpleInterceptor provides before and after execution hooks that you can override. We can create a simple timing interceptor by overriding the hooks and writing to the Trace object:
``` csharp
public class TimingInterceptor : SimpleInterceptor
{
    readonly Stopwatch _stopwatch = new Stopwatch();
 
    protected override void BeforeInvoke(IInvocation invocation)
    {
        _stopwatch.Start();
    }
 
    protected override void AfterInvoke(IInvocation invocation)
    {
        _stopwatch.Stop();
        string message = string.Format( "Execution of {0} took {1}.",
                                        invocation.Request.Method,
                                        _stopwatch.Elapsed );
        Trace.WriteLine(message);
        _stopwatch.Reset();
    }
}
```
There is a lot more that can be done with the IInvocation object. You should really take a minute to look over the interface and see some of the information available. I have also included IProxyRequest as there is a lot of good information there accessible from IInvocation:
``` csharp
public interface IInvocation
{
    /// <summary>
    /// Gets the request, which describes the method call.
    /// </summary>
    IProxyRequest Request { get; }
 
    /// <summary>
    /// Gets the chain of interceptors that will be executed before the target method is called.
    /// </summary>
    IEnumerable<IInterceptor> Interceptors { get; }
 
    /// <summary>
    /// Gets or sets the return value for the method.
    /// </summary>
    object ReturnValue { get; set; }
 
    /// <summary>
    /// Continues the invocation, either by invoking the next interceptor in the chain, or
    /// if there are no more interceptors, calling the target method.
    /// </summary>
    void Proceed();
}

public interface IProxyRequest
{
    /// <summary>
    /// Gets the kernel that created the target instance.
    /// </summary>
    IKernel Kernel { get; }
 
    /// <summary>
    /// Gets the context in which the target instance was activated.
    /// </summary>
    IContext Context { get; }
 
    /// <summary>
    /// Gets or sets the proxy instance.
    /// </summary>
    object Proxy { get; set; }
 
    /// <summary>
    /// Gets the target instance.
    /// </summary>
    object Target { get; }
 
    /// <summary>
    /// Gets the method that will be called on the target instance.
    /// </summary>
    MethodInfo Method { get; }
 
    /// <summary>
    /// Gets the arguments to the method.
    /// </summary>
    object[] Arguments { get; }
 
    /// <summary>
    /// Gets the generic type arguments for the method.
    /// </summary>
    Type[] GenericArguments { get; }
 
    /// <summary>
    /// Gets a value indicating whether the request has generic arguments.
    /// </summary>
    bool HasGenericArguments { get; }
}
```
Given this information you can implement simple or more complex interceptors capable of utilizing a large amount of information to do the job.

#### ActionInterceptor
The ActionInterceptor takes an Action<IInvocation> in its .ctor
``` csharp
ActionInterceptor interceptor =
    new ActionInterceptor( invocation => Console.WriteLine("Executing {0}.", invocation.Request.Method) );
```
If you want a quick spike and don’t want to define a full interceptor, the ActionInterceptor is very handy.

### Multiple Interceptors

You have seen how to attach interceptors via a static registry (IKernel extensions for specific methods) and attaching them using bindings. Now, what may have not been obvious, is that you can attach multiple interceptors to a given binding.
``` csharp
[Fact]
public void CanAttachMultipleInterceptors()
{
    using (StandardKernel kernel = CreateDefaultInterceptionKernel())
    {
        var binding = kernel.Bind<FooImpl>().ToSelf();
        binding.Intercept().With<FlagInterceptor>();
        binding.Intercept().With<CountInterceptor>();
 
        var foo = kernel.Get<FooImpl>();
 
        Assert.False(FlagInterceptor.WasCalled);
        Assert.Equal(0, CountInterceptor.Count);
 
        foo.Foo(); // call method to invoke interceptors
 
        Assert.True(FlagInterceptor.WasCalled);
        Assert.Equal(1, CountInterceptor.Count);
    }
}
```
We can also change the order in which the interceptors are executed:
``` csharp
[Fact]
public void CanAttachMultipleInterceptors()
{
    using (StandardKernel kernel = CreateDefaultInterceptionKernel())
    {
        var binding = kernel.Bind<FooImpl>().ToSelf();
        binding.Intercept().With<FlagInterceptor>().InOrder(2);
        binding.Intercept().With<CountInterceptor>().InOrder(1);
        binding.Intercept().With<TimingInterceptor>().InOrder(3);
        // ...
    }
}
```
If you do not want to use the binding syntax to specify interceptors, you can use attributes to mark specific classes for interception. The [Intercept] attribute is an abstract attribute for you to extend and use on classes and methods. When a derived attribute is applied on a class, it will register interception on all virtual methods of that class. If you wish to disable interception for a particular method, you can apply the [DoNotIntercept] attribute. If you only want to intercept a particular method in a class, just apply the derived attribute to that single method.

You must extend the base [Intercept] attribute as it needs to know what interceptor to attach. You specify this by overriding the CreateInterceptor method. We can use the TimingInterceptor used previously to create a  attribute and apply it to a simple class.
``` csharp
public class TimeAttribute : InterceptAttribute
{
    public override IInterceptor CreateInterceptor( IProxyRequest request )
    {
        return request.Context.Kernel.Get<TimingInterceptor>();
    }
}
 
public class ObjectWithMethodInterceptor
{
    [Time] // intercepted
    public virtual void Foo()
    {
    }
 
    // not intercepted
    public virtual void Bar()
    {
    }
}
 
[Time]
public class ObjectWithClassInterceptor
{
    // intercepted
    public virtual void Foo()
    {
    }
 
    [DoNotIntercept] // not intercepted
    public virtual void Bar()
    {
    }
 
    // not intercepted - method is not virtual
    public void Baz()
    {
    }
}
```
There is one other part of the interception attributes that is also supported from the binding configuration: Order. When you attach interception attributes, you can specify the order of the interceptor. You do not need to specify this in your custom attributes; it is handled by the injection system. All you have to do is specify the order in the attribute usage.
``` csharp
[Count(Order = 1)]
public class ObjectWithOrderedInterceptors
{
    [Flag(Order = 0)]
    public virtual void Foo()
    {
    }
 
    public virtual void Baz()
    {
    }
}
```
One thing you may have noticed is that I used two different interceptor types. This is perfectly valid. All virtual calls of ObjectWithOrderedInterceptors will be counted. In addition, calls to Foo will also set a flag and the flag will be set before the count interceptor has a chance to run.

This can be important if you are trying to time calls with an interceptor; yes, the interception system is going to skew your results either way, but the amount you are over can vary drastically if your timing interceptor is first or last in the chain.

### Conclusion

That pretty much covers the Ninject.Extensions.Interception interceptors. Using this foundation, you can create just about any type of interception scheme you can think of.