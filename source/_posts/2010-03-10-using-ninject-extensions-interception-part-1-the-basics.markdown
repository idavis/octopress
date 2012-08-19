---
layout: post
title: "Using Ninject.Extensions.Interception Part 1 : The Basics"
date: 2010-03-10 17:21
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
### About the Extension

The Ninject interception extension relies on the DynamicProxy implementations of Castle and LinFu to generate proxy instances. The interception extension is essentially an abstraction of the proxying components leveraging Ninject’s feature set along with interception utilities.

### Enabling the Extension

There are two ways to enable the extension. The easiest way is to use Ninject’s automatic extension loading mechanism. By default Ninject will load all modules from assemblies that match the pattern Ninject.Extensions.*.dll in the same directory as the Ninject assembly. This works fine except that we have multiple options for the DynamicProxy implementation.

The .NET 3.5 release contains both the LinFuModule and the DynamicProxy2Module. If you have automatic extension loading enabled, it will load both modules. To make it easier, there are two other .NET 3.5 releases that either contain the LinFuModule, or the DynamicProxy2Module. All other platform releases only have one module: Mono->LinFu, Silverlight 3->Castle.

If you have automatic extension loading disabled, there is no need to worry, you can simply pass one of the modules into the kernel .ctor as usual. Now that we have the extension loaded, it is time to start using the extension.

### Extensibility Provided

There are three main ways that that you can use the extension:

Interception on the Binding
Method Interception
Facilities Provided by the Extension
All of the binding capability of the extension are based on extension methods on IKernel and IBindingSyntax. The extensions for IKernel provide method interception where you can intercept very specific calls when instances of a type are requested. The IBindingSyntax extensions provide class level interception using a specific IInterceptor implementation.

## *** Important Note ***

Your methods/properties to be intercepted must be virtual! The DynamicProxy implementations are still bound by inheritance rules. If you look at all examples in this series, all intercepted members are virtual.

### Interception on the Binding

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
In order to provide class level interception, you must create a binding.
``` csharp
kernel.Bind<ViewModel>().ToSelf().Intercept().With<FlagInterceptor>();
```
The Intercept() extension creates a hint inside the kernel that when this binding is used, that we need to look into the proxying strategies. The With<T>() call declares the IInterceptor to apply with this binding. You can also supply an instance in the With<T>() call in order to pass it a configured interceptor.
``` csharp
ActionInterceptor interceptor =
    new ActionInterceptor( invocation => Console.WriteLine("Executing!") );
kernel.Bind<ViewModel>().ToSelf().Intercept().With( interceptor );
```
### Method Interception

Using method interception, for a given method or property, you can replace, wrap, prepend, or append functionality. Remember, these are provided as extension methods on IKernel giving you the following calls from an IKernel instance:
``` csharp
IAdviceTargetSyntax Intercept( Predicate<IProxyRequest> predicate )
 
void AddMethodInterceptor( MethodInfo method, Action<IInvocation> action )
 
void InterceptReplace<T>( Expression<Action<T>> methodExpr,
        Action<IInvocation> action )
 
void InterceptAround<T>( Expression<Action<T>> methodExpr,
       Action<IInvocation> beforeAction,
       Action<IInvocation> afterAction )
 
void InterceptBefore<T>( Expression<Action<T>> methodExpr,
       Action<IInvocation> action )
 
void InterceptAfter<T>( Expression<Action<T>> methodExpr,
       Action<IInvocation> action )
 
void InterceptReplaceGet<T>( Expression<Func<T, object>> propertyExpr,
       Action<IInvocation> action )
 
void InterceptAroundGet<T>( Expression<Func<T, object>> propertyExpr,
       Action<IInvocation> beforeAction,
       Action<IInvocation> afterAction )
 
void InterceptBeforeGet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> action )
 
void InterceptAfterGet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> action )
 
void InterceptReplaceSet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> action )
 
void InterceptAroundSet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> beforeAction,
        Action<IInvocation> afterAction )
 
void InterceptBeforeSet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> action )
 
void InterceptAfterSet<T>( Expression<Func<T, object>> propertyExpr,
        Action<IInvocation> action )
```
For example:
``` csharp
public class Foo : IFoo
{
    public virtual void ThrowsAnError()
    {
        throw new Exception();
    }
}
 
public class MethodTests
{
    public void NoOpASingleMethod()
    {
        Kernel.Bind<Foo>().ToSelf();
        Kernel.InterceptReplace<Foo>(foo => foo.ThrowsAnError(), invocation => {} );
        var foo = Kernel.Get<Foo>();
        foo.ThrowsAnError(); // Nothing happens
    }
}
```
One key to note is that the interception is for the implementing type, not for the service that the bindings are configured for. For example, there will be no interceptor attached when we configure the interception on the interface:
``` csharp
public void NoInterceptionPerformed()
{
    Kernel.Bind<IFoo>().To<Foo>();
    Kernel.InterceptReplace<IFoo>(foo => foo.ThrowsAnError(), invocation => {} );
    var foo = Kernel.Get<IFoo>();
    foo.ThrowsAnError(); // exception is thrown as usual
}
```
### Facilities Provided by the Extension

The extension provides a couple of utility mechanisms to aid programmers. As mention the a previous blog post, the extension supplies an automatic INotifyPropertyChanged implementation for your view model classes.

If you also take a look back to the block posts on weak proxies and weak proxy factories, this functionality is currently being worked into the extension and will be presented in a future blog post.

### Conclusion

Now that we have covered the basics with the interception extension, we can move into more advanced topics such at attaching multiple proxies, proxy ordering, and writing custom interceptors in the next post.