---
layout: post
title: "Conventions based binding with Ninject 2.0"
date: 2009-09-24 16:28
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

Conventions binding is implemented through extension methods on the IKernel interface. In order to use these extensions, first download or compile a release of the Ninject.Extensions.Conventions project. Once the library is referenced, type:
``` csharp
using Ninject.Extensions.Conventions;
```
into the source file you are using to configure your Ninject kernel. This will add two additional methods to the IKernel interface:
``` csharp
void Scan(AssemblyScanner assemblyScanner);
void Scan(Action<Assemblyscanner> scan);
```
You can either construct and make calls on an AssemblyScanner, or you can pass a function/lambda expression that will run the same operations. This implementation is based on [StructureMap 2.5](http://structuremap.sourceforge.net/Default.htm) AssemblyScanner by [Jeremy Miller](http://codebetter.com/blogs/jeremy.miller). Here are two tests showing the same bindings, but using different calls.
``` csharp
[Fact]
public void UsingDefaultConventionsResolvesCorrectly()
{
    IKernel kernel = new StandardKernel();
    var scanner = new AssemblyScanner();
    scanner.Assembly( Assembly.GetExecutingAssembly() );
    scanner.UsingDefaultConventions();
    kernel.Scan( scanner );
    var instance = kernel.Get<IDefaultConvention>();
    Assert.NotNull( instance );
    Assert.Equal( typeof (DefaultConvention), instance.GetType() );
}
 
[Fact]
public void TestBindingGeneratorInLambaSyntax()
{
    IKernel kernel = new StandardKernel();
    kernel.Scan( x =>
                 {
                     x.Assembly( Assembly.GetExecutingAssembly() );
                     x.Using<DefaultBindingGenerator>();
                 }
        );
    var instance = kernel.Get<IDefaultConvention>();
    Assert.NotNull( instance );
    Assert.Equal( typeof (DefaultConvention), instance.GetType() );
}
```
The x.Using() call takes an IBindingGenerator or you can pass it an instance: x.Using(new DefaultBindingGenerator()). Binding generators process types and add kernel bindings. For instance, the DefaultBindingGenerator processes types and looks for interfaces that match I[TypeName]. If a match is found, it binds I[TypeName] to the implementation TypeName. There are other binding generators that can use regex patterns to bind classes and to close generic interfaces:
``` csharp
[Fact]
public void CanResolveMatchAtEndOfInterface()
{
    var regexBindingGenerator =
        new RegexBindingGenerator( "(I)(?<name>.+)" );
    IKernel kernel = new StandardKernel()
    regexBindingGenerator.Process( typeof (DefaultView), kernel );
    Assert.IsType(typeof(DefaultView), kernel.Get<IDefaultConvention>());
}
 
[Fact]
public void OpenGenericsAreFound()
{
    IKernel kernel = new StandardKernel();
    kernel.Scan(
        x =>
        {
            x.CallingAssembly();
            x.Using(
                new GenericBindingGenerator( typeof (IGenericView<>) ) );
        } );
    object target = kernel.Get<IGenericView<IDefaultConvention>>();
    Assert.IsAssignableFrom<DefaultConventionView>( target );
    target = kernel.Get<IGenericView<string>>();
    Assert.IsAssignableFrom<StringView>( target );
}
```
There is really too much to talk about in a single post, but in looking through the IAssemblyScanner interface, you should get a decent idea of what it can do. The calls break down into: determine which assemblies to use, include or exclude types and namespaces from being candidates, automatically load Ninject modules, and run binding generators against all types being processed.
``` csharp
public interface IAssemblyScanner
{
    void Assembly( Assembly assembly );
    void CallingAssembly();
    void Assemblies( IEnumerable<Assembly> assemblies );
    void Assembly( string assembly );
    void Assemblies( IEnumerable<string> assemblies );
    void Assemblies( IEnumerable<string> assemblies, Predicate<Assembly> filter );
    void AssemblyContaining<T>();
    void AssemblyContaining( Type type );
    void AssemblyContaining( IEnumerable<Type> types );
    void AssembliesFromPath( string path );
    void AssembliesFromPath( string path, Predicate<Assembly> assemblyFilter );
    void AssembliesMatching( string pattern );
    void AssembliesMatching( IEnumerable<string> patterns );
    void IncludeType<T>();
    void IncludeType( Type type );
    void IncludeTypes( IEnumerable<Type> filters );
    void IncludeTypes( Predicate<Type> filter );
    void IncludeNamespace( string nameSpace );
    void IncludeNamespaceContainingType<T>();
    void ExcludeType<T>();
    void ExcludeType( Type type );
    void ExcludeTypes( IEnumerable<Type> filters );
    void ExcludeTypes( Predicate<Type> filter );
    void ExcludeNamespace( string nameSpace );
    void ExcludeNamespaceContainingType<T>();
    void Using<T>() where T : IBindingGenerator, new();
    void Using( IBindingGenerator generator );
    void UsingDefaultConventions();
    void AutoLoadModules();
    void IncludeAllTypesOf<T>();
    void IncludeAllTypesOf( Type type );
}
``` 
When processing assembly filters, a temporary AppDomain is spawned for the assembly processing so that if the predicate fails, we don’t have to worry about unneeded assemblies staying loaded in our application.

This has been a brief introduction to the conventions based binding extension for [Ninject 2.0](http://github.com/ninject/ninject). I hope it is helpful in your configuration of the Ninject kernel.