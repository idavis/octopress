---
layout: post
title: "Weak Proxies"
date: 2009-07-27 15:53
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
I wanted my proxy factory to hand out instances, but clean up the instance as soon as the caller no longer needs the service. I also didn’t want the caller running the cleanup code. This allows me to pool proxy instances and clean them up as I see fit.

There is an inherent issue with this idea: how do I know when the caller is done with the proxy? The answer, while not straight forward, is pretty cool. To start off, we will only consider what we hand off to the caller when the service is requested.

Let’s assume we have and IFoo contract and a Foo service that can be consumed. We want the caller to rely on the contract while we deal with the cleanup. Here is a very simple example.
``` csharp
public interface IFoo
{
    void Bar();
}
 
public class Foo : IFoo, IDisposable
{
    public bool IsDisposed { get; private set; }
 
    #region IDisposable Members
 
    public void Dispose()
    {
        IsDisposed = true;
    }
 
    #endregion
 
    #region IFoo Members
 
    public void Bar()
    {
        Console.WriteLine( &quot;Foo::Bar&quot; );
    }
 
    #endregion
}
```
Now we want to hand out IFoo, and allow the caller to release control of the service. This is done most simply in two ways. First, use the pointer to the IFoo service and then go out of scope. Second, hold the IFoo pointer in a member variable and set it to null when done. For the sake of the example, we are going to play with the second option.
``` csharp
public class Baz
{
    private IFoo _foo;
 
    public Baz( IFoo foo )
    {
        _foo = foo;
    }
 
    public void Action()
    {
        _foo.Bar();
    }
 
    public void Release()
    {
        _foo = null;
    }
}
``` 
We can supply the IFoo into the Baz instance via dependency injection. When we are done with the IFoo instance, we set the pointer to null, or the GC will do it for us when the object goes out of scope. This gives us a very simple model from which things get a lot more fun.

Weak<T> is the base for this post. It is a WeakReference with strongly typed access to the target instance. This can be optimized, but is much more understandable as-is.
``` csharp
public class Weak<T> : WeakReference
{
    public Weak( T target )
        : base( target )
    {
    }
 
    protected Weak( T target, bool trackResurrection )
        : base( target, trackResurrection )
    {
    }
 
    protected Weak( SerializationInfo info, StreamingContext context )
        : base( info, context )
    {
    }
 
    public T Instance
    {
        get { return (T) Target; }
    }
}
```
We can extend this class to create a decorator proxy based on the weak reference to the real service:
``` csharp
public class WeekFoo : Weak<IFoo>, IFoo
{
    public WeekFoo( IFoo target )
        : base( target )
    {
    }
 
    protected WeekFoo( IFoo target, bool trackResurrection )
        : base( target, trackResurrection )
    {
    }
 
    protected WeekFoo( SerializationInfo info, StreamingContext context )
        : base( info, context )
    {
    }
 
    #region IFoo Members
 
    void IFoo.Bar()
    {
        Instance.Bar();
    }
 
    #endregion
}
```  
Now we have an object with a weak reference base that behaves like an IFoo. There is a drawback to this which should be pretty obvious: you have to generate a proxy for every service you want to track and hand-write every method in the interface. This is where a dynamic proxy framework like Castle.DynamicProxy2 or LinFu.DynamicProxy can save us a lot of time. For the example, I am going to use LinFu. Really, all we want is a wrapper that will use the underlying weak object to make the interface calls for us. We can also set it up to throw if we have released the weak reference.
``` csharp
public class WeakInterceptor<T> : IInvokeWrapper
{
    private readonly Weak<T> _target;
 
    public WeakInterceptor( Weak<T> target )
    {
        _target = target;
    }
 
    #region IInvokeWrapper Members
 
    public void BeforeInvoke( InvocationInfo info )
    {
    }
 
    public object DoInvoke( InvocationInfo info )
    {
        T instance = _target.Instance;
        if ( _target.IsAlive )
        {
            return info.TargetMethod.Invoke( instance, info.Arguments );
        }
        throw new InvalidOperationException();
    }
 
    public void AfterInvoke( InvocationInfo info, object returnValue )
    {
    }
 
    #endregion
}
``` 
This is a much more elegant approach and requires very little work to set up a full interface proxy around the weak reference for each service we want to wrap. Finally, we can test the code out to make sure our assumptions are working correctly.
``` csharp
internal class Program
{
    private static void Main( string[] args )
    {
        LinFu();
        ManualProxy();
        Console.ReadLine();
    }
 
    public static void LinFu()
    {
        var foo = new Foo();
        var weakReference = new Weak<WeakInterceptor<IFoo>>(
            new WeakInterceptor<IFoo>;(
                new Weak<IFoo>( foo ) ) );
        var proxyFactory = new ProxyFactory();
        var baz = new Baz( proxyFactory.CreateProxy<IFoo>( weakReference.Instance ) );
 
        Verify( foo, weakReference, baz );
    }
 
    private static void ManualProxy()
    {
        var foo = new Foo();
        var weakReference = new Weak<WeekFoo>( new WeekFoo( foo ) );
        var baz = new Baz( weakReference.Target as IFoo );
 
        Verify( foo, weakReference, baz );
    }
 
    private static void Verify( Foo foo, WeakReference weakReference, Baz baz )
    {
        baz.Action();
        GC.Collect();
 
        Debug.Assert( weakReference.IsAlive );
        baz.Release();
        GC.Collect();
 
        Debug.Assert( weakReference.IsAlive == false );
        foo.Dispose();
        Debug.Assert( foo.IsDisposed );
    }
}
``` 
We now have two methods for handing out weak service references to clients and allowing us to control the lifetime of the services for pooling, disposal, or whatever else we want to do without the client having to deal with the details. This is just a basic example. You can easily extend this into a service factory/container and track the instances you hand out.