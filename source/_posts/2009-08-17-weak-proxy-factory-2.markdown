---
layout: post
title: "Weak Proxy Factory"
date: 2009-08-17 15:59
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
In the previous article I showed how one could construct a weak proxy object using LinFu. This idea very slick, but it doesn’t do us much good unless there is something built around it.

DISCLAIMER: Blocks of this code are taken directly from the Ninject 2.0 source base and re-purposed. All code in this article is released under the same licenses as the source works (Apache 2 and MS-PL).

To start, we have IWeakCache (formerly [ICache.cs](http://github.com/ninject/ninject/blob/de9a243fc8eff2aeb91cd763db5dcdcd5bd6fe74/src/Ninject/Activation/Caching/ICache.cs)) and [ICachePruner.cs](http://github.com/ninject/ninject/blob/de9a243fc8eff2aeb91cd763db5dcdcd5bd6fe74/src/Ninject/Activation/Caching/ICachePruner.cs) to describe our cache backing and the cleanup strategy. The IWeakCache is like the Ninject ICache except that I don’t provide the Get method and it takes a Weak> pointer and the source object. We can watch the weak reference for cleanup and execute it on the original object.
``` csharp
public interface IWeakCache
{
    void Remember<T>( Weak<WeakInterceptor<T>> reference, T instance );
 
    void Prune();
 
    void Clear();
}
 
public interface ICachePruner
{
    void Start( IWeakCache cache );
 
    void Stop();
}
```
I use the [GarbageCollectionCachePruner.cs](http://github.com/ninject/ninject/blob/de9a243fc8eff2aeb91cd763db5dcdcd5bd6fe74/src/Ninject/Activation/Caching/GarbageCollectionCachePruner.cs) directly from Ninject but change the ICache and ICachePruner interfaces to my own.

Given that I need to create my own cache to monitor weak references, I modified the [Cache.cs](http://github.com/ninject/ninject/blob/de9a243fc8eff2aeb91cd763db5dcdcd5bd6fe74/src/Ninject/Activation/Caching/Cache.cs) file to accommodate this.
``` csharp
public class WeakProxyCache : IDisposable, IWeakCache
{
    private readonly Multimap<Type, CacheEntry> _entries = new Multimap<Type, CacheEntry>();
 
    #region Implementation of IWeakCache
 
    public void Remember<T>( Weak<WeakInterceptor<T>> reference, T instance )
    {
        lock ( _entries )
        {
            CacheEntry entry = CacheEntry.FromObject( reference, instance );
            _entries[entry.Service].Add( entry );
        }
    }
 
    public void Prune()
    {
        lock ( _entries )
        {
            List<CacheEntry> expiredValues = _entries
                .SelectMany( entry => entry.Value )
                .Where( item => !item.Scope.IsAlive )
                .ToList();
            expiredValues.ForEach( Forget );
        }
    }
 
    public void Clear()
    {
        lock ( _entries )
        {
            _entries
                .SelectMany( e => e.Value )
                .ToList()
                .ForEach( entry => _entries[entry.Service]
                                       .Remove( entry ) );
        }
    }
 
    #endregion
 
    private void Forget( CacheEntry entry )
    {
        _entries[entry.Service].Remove( entry );
        var disposable = entry.Instance as IDisposable;
        if ( disposable != null )
        {
            disposable.Dispose();
        }
    }
 
    #region Implementation of IDisposable
 
    protected bool IsDisposed { get; private set; }
 
    public void Dispose()
    {
        Dispose( true );
        GC.SuppressFinalize( this );
    }
 
    public void Dispose( bool disposing )
    {
        if ( disposing && !IsDisposed )
        {
            Clear();
            IsDisposed = true;
        }
    }
 
    #endregion
 
    #region Nested type: CacheEntry
 
    protected class CacheEntry
    {
        public WeakReference Scope { get; private set; }
        public Type Service { get; private set; }
        public object Instance { get; private set; }
 
        public static CacheEntry FromObject<T>( WeakReference reference, T instance )
        {
            return new CacheEntry
                   {
                       Scope = reference,
                       Instance = instance,
                       Service = typeof (T)
                   };
        }
    }
 
    #endregion
}
```
With this, I now have everything I need to create my WeakProxyFactory. I set up the .ctor to take an IWeakCache and ICachePruner so that we can adjust the storage and cleanup as needed. In this class, I am actually using the CommonServiceLocator to get the real object instance.
``` csharp
public class WeakProxyFactory
{
    private readonly IWeakCache _defaultProxyCache;
    private readonly ICachePruner _defaultProxyCachePruner;
    private readonly ProxyFactory _proxyFactory;
 
    public WeakProxyFactory()
        : this( new WeakProxyCache(), new GarbageCollectionCachePruner() )
    {
    }
 
    public WeakProxyFactory( IWeakCache cache, ICachePruner cachePruner )
    {
        _proxyFactory = new ProxyFactory();
        _defaultProxyCache = cache;
        _defaultProxyCachePruner = cachePruner;
        _defaultProxyCachePruner.Start( _defaultProxyCache );
    }
 
    public virtual T GetProxy<T>()
    {
        var instance = GetInstance<T>();
 
        T proxy = WrapInstance( instance );
 
        return proxy;
    }
 
    protected virtual T WrapInstance<T>( T instance )
    {
        var weakReference = new Weak<WeakInterceptor<T>>(
            new WeakInterceptor<T>(
                new Weak<T>( instance ) ) );
 
        _defaultProxyCache.Remember( weakReference, instance );
 
        var proxy = _proxyFactory.CreateProxy<T>( weakReference.Instance );
        return proxy;
    }
 
    protected virtual T GetInstance<T>()
    {
        return ServiceLocator.Current.GetInstance<T>();
    }
}
```
Now you can create a weak proxy simply by using the factory, or setting it up in an IoC container. To use the previous example of hooking IFoo to Foo, and injecting the weak reference into Baz objects, I can configure my application. This is going to look like a lot, but I am just being overly verbose showing how everything is set up.
``` csharp
public class WeakProxyProvider<T> : Provider<T>
{
    #region Overrides of Provider<T>
 
    protected override T CreateInstance( IContext context )
    {
        return context.Kernel.Get<WeakProxyFactory>().GetProxy<T>();
    }
 
    #endregion
}

// ....

public static void Configure()
{
    IKernel kernel = new StandardKernel();
    kernel.Bind( typeof (WeakProxyProvider<>) ).ToSelf().InSingletonScope();
    kernel.Bind<IFoo>().ToProvider<WeakProxyProvider<Foo>>().InTransientScope();
    kernel.Bind<Foo>().ToSelf().InTransientScope();
    kernel.Bind<Baz>().ToSelf().InTransientScope();
    kernel.Bind<IWeakCache>().To<WeakProxyCache>().InSingletonScope();
    kernel.Bind<ICachePruner>().To<GarbageCollectionCachePruner>()
        .WithConstructorArgument( "pruneInterval", TimeSpan.FromSeconds( 1 ) );
    kernel.Bind<WeakProxyCache>().ToSelf();
    kernel.Bind<WeakProxyFactory>().ToSelf().InSingletonScope();
 
    var ninjectServiceLocator = new NinjectServiceLocator( kernel );
    ServiceLocator.SetLocatorProvider( () => ninjectServiceLocator );
}
```
So after all of this work, what does it buy me? My Baz objects now have IFoo weak proxy instances automatically injected into them when I make requests to the ServiceLocator. Once I null the pointer to the Foo object, the cache pruner will clean things up:
``` csharp
public static class SystemTest
{
    private const int size = 6;
 
    public static void Run()
    {
        Configure();
        var bazs = new List<Baz>();
        for ( int i = 0; i < size; i++ )
        {
            bazs.Add( ServiceLocator.Current.GetInstance<Baz>() ); // Where the magic happens
        }
 
        for ( int i = 0; i < size; i++ )
        {
            Baz baz = bazs[i];
            Console.WriteLine( "Foo Id: {0}", baz.Foo.Id );
            baz.Action();
            GC.Collect( 2 );
 
            baz.Release();
            GC.Collect( 2 ); // The Foo instances are disposed of by the system for you.
            Thread.Sleep( 100 );
        }
    }
}
```
So, we have now seen a full implementation of a weak proxy factory handing out weak substitutes for the real objects. But what can we do from here? Is there anything else? YES! Now what we have full control over the real object lifetime, we can implement pooling. So, instead of handing out pointers to new weak objects, I can reuse objects in my pool, or use a lock to wait until a resource is available. These are both potentially big projects, so I am going to keep it small for now and just show a very minimalist example.

He is a small example that will let you set the size of the pool and it will reuse instances. Every time an instance is requested, we will clean the dead objects from the pool and refill if needed.
``` csharp
public class PoolingWeakProxyProvider<T> : Provider<T>
{
    private int _currentIndex;
    private List<Weak<T>> _pool = new List<Weak<T>>();
 
    public PoolingWeakProxyProvider()
        : this( 5 )
    {
    }
 
    private PoolingWeakProxyProvider( int poolSize )
    {
        PoolSize = poolSize;
        _pool = new List<Weak<T>>();
    }
 
    #region Overrides of Provider<T>
 
    protected override T CreateInstance( IContext context )
    {
        lock ( _pool )
        {
            Prune();
            var proxyFactory = context.Kernel.Get<WeakProxyFactory>();
            if ( _pool.Count < PoolSize )
            {
                var item = proxyFactory.GetProxy<T>();
                _pool.Add( new Weak<T>( item ) );
                return item;
            }
            else
            {
                return _pool[( _currentIndex++ ) % PoolSize].Instance;
            }
        }
    }
 
    #endregion
 
    public int PoolSize { get; private set; }
 
    public void Prune()
    {
        lock ( _pool )
        {
            var expired = _pool
                .Where( item => !item.IsAlive )
                .ToList();
            expired.ForEach( item => _pool.Remove( item ) );
        }
    }
}
```
Going back to our IoC container configuration (Ninject), we can put these two lines in:
``` csharp
public static void Configure()
{
...
    kernel.Bind( typeof (PoolingWeakProxyProvider<>) ).ToSelf().InSingletonScope();
    kernel.Bind<IFoo>().ToProvider<PoolingWeakProxyProvider<Foo>>().InTransientScope();
...
}
```
I think that this can potentially be very useful in dealing with items such as WCF proxies. You could keep a pool and trim proxies as they age, and return them to requesters if there is no live usage, but the connection is still good.