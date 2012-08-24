---
layout: post
title: "Safe-Dereference / Inverse Null Coalescing Support for C#"
date: 2012-08-23 16:59
comments: true
categories: [C#, groovy, expressions trees]
---
Groovy has a very handy safe-dereference operator `?.` which allows for nested dereferencing without having to worry about a null reference exception.

``` groovy
people << new Person(name:'Ian')
highestZipCode = people.collect{ p -> p.Address?.ZipCode }.max()
println highestZipCode
```

While we cannot add custom operators to C#, we can use extension methods and expressions to simulate `p -> p.Address?.ZipCode`. To follow the naming convention in LINQ, I am going to create an extension method on `object` called `ValueOrDefault`. Since operators are not an option, expression trees are the primary option. Assuming we have a an object, our goal is to support `target.ValueOrDefault( item => item.Foo.Bar().Baz )` evaluating the expression layer by layer until we finish or encounter a `default(T)` value. To evaluate the expression, we have to parse from the outside in, but evaluate from the inside out, and wrap every subexpression evaluation in the same safe call to `ValueOrDefault`. The recursion is rather fun:

```
Start              => ValueOrDefault

ValueOrDefault     => TerminalValue
ValueOrDefault     => EvaluateExpression

EvaluateExpression => ValueOrDefault
EvaluateExpression => EvaluateMember
EvaluateExpression => TerminalValue

EvaluateMember     => EvaluateMember
EvaluateMember     => ValueOrDefault
```

In order to evaluate `target.ValueOrDefault( item => item.Foo.Bar().Baz )` we need to

1.  Determine if `target` is null, if so, return null
2.  Pass `target` as the instance and `item => item.Foo.Bar().Baz`
3.  Pull off the `.Baz` expression and evaluate `item.Foo.Bar()` with `target` as the instance
4.  Pull off the `.Bar()` and evaluate `item.Foo` with `target` as the instance
5.  Pull off the `.Foo` and evaluate `item` with `target` as the instance
6.  There are no more member or method expressions, so `target` is passed back up the stack
7.  `.Foo` is evaluated on the `target` instance. If `Foo` evaluates to `default(T)` we cascade back up the recursion tree and stop evaluating. If not, we proceed to #8 passing the value returned by `.Foo` as the `target` instance.
8.  `.Bar()` is executed on the `target` instance. Same behavior as #7, pass the return `value` back up the stack
9.  `.Baz` is evaluated on the `target` instance returned from `.Bar()`

Well, that's enough theory, let's look at the implementation.

``` csharp
using System;
using System.Linq.Expressions;
using System.Reflection;

public static class ObjectExtensions
{
  public static TValue ValueOrDefault<TSource, TValue>
                         ( this TSource instance,
                           Expression<Func<TSource, TValue>> expression )
  {
    return ValueOrDefault( instance, expression, true );
  }

  private static TValue ValueOrDefault<TSource, TValue>
                          ( this TSource instance,
                            Expression<Func<TSource, TValue>> expression,
                            bool nested )
  {
      return ReferenceEquals( instance, default( TSource ) )
                 ? default( TValue )
                 : nested ? EvaluateExpression( instance, expression )
                          : expression.Compile()( instance );
  }

  internal static TProperty EvaluateExpression<TSource, TProperty>
                              ( TSource source,
                                Expression<Func<TSource, TProperty>> expression )
  {
    var method = expression.Body as MethodCallExpression;
    if ( method != null ) {
      return ValueOrDefault( source, expression, false );
    }

    var body = expression.Body as MemberExpression;
    if ( body == null ) {
      const string format = "Expression '{0}' must refer to a property.";
      string message = string.Format( format, expression );
      throw new ArgumentException( message );
    }

    object value = EvaluateMemberExpression( source, body );
    if ( ReferenceEquals( value, null ) ) {
      return default( TProperty );
    }
    return (TProperty) value;
  }

  private static object EvaluateMemberExpression
                          ( object instance,
                            MemberExpression memberExpression )
  {
    if ( memberExpression == null ) {
      return instance;
    }
    instance = EvaluateMemberExpression( instance, memberExpression.Expression as MemberExpression );
    var propertyInfo = memberExpression.Member as PropertyInfo;
    instance = ValueOrDefault( instance, item => propertyInfo.GetValue( item, null ), false );
    return instance;
  }
}
```

The key thing to grok is the inside out evaluation. If you want to play around with this code, I have included a number of tests below to make sure that everything is in working order. Having `?.` added to the C# language is high on my list of desired vNext features. Hopefully the C# team agrees.

``` csharp Tests and Example Usage
using Xunit;

public class Person
{
  public string Name { get; set; }
  public Address Address { get; set; }
}

public class Address
{
  public string StreetName { get; set; }
  public ZipCode ZipCode { get; set; }
}

public class ZipCode
{
  public int Body { get; set; }
  public int? Suffix { get; set; }
  public Zone Zone { get; set; }
  public Zone GetZone() { return Zone; }
}

public class Zone
{
  public string Name { get; set; }
}

public class ObjectExtensionTests
{
  [Fact]
  public void when_the_accessed_object_is_null_then_null_is_returned()
  {
    Person person = null;
    string result = person.ValueOrDefault( p => p.Name );
    Assert.Equal( null, result );
  }

  [Fact]
  public void when_the_accessed_property_is_null_then_null_is_returned()
  {
    var person = new Person { Name = null };
    string value = person.ValueOrDefault( p => p.Name );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_the_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Ian";
    var person = new Person { Name = name };
    string value = person.ValueOrDefault( p => p.Name );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_complex_property_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    Address value = person.ValueOrDefault( p => p.Address );
    Assert.Equal( null, value );
  }

  [Fact]
  public void EvaluateExpression_non_nested_properties_are_evaluated()
  {
     const string name = "Ian";
     var person = new Person { Name = name };
     string value = ObjectExtensions.EvaluateExpression( person, p => p.Name );
     Assert.Equal( name, value );
  }

  [Fact]
  public void EvaluateExpression_nested_properties_are_evaluated()
  {
    const string name = "Ian";
    var person = new Person { Address = new Address { StreetName = name } };
    string value = ObjectExtensions.EvaluateExpression( person, p => p.Address.StreetName );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_nested_accessed_property_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    string value = person.ValueOrDefault( p => p.Address.StreetName );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_a_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Ian";
    var person = new Person { Address = new Address { StreetName = name } };
    string value = person.ValueOrDefault( p => p.Address.StreetName );
    Assert.Equal( name, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_propertys_parent_is_null_then_defaultT_is_returned()
  {
    var person = new Person { Address = null };
    int value = person.ValueOrDefault( p => p.Address.ZipCode.Body );
    Assert.Equal( 0, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_propertys_parent_is_null_then_defaultT_is_returned_for_nullable_types()
  {
    var person = new Person { Address = null };
    int? value = person.ValueOrDefault( p => p.Address.ZipCode.Suffix );
    Assert.Equal( null, value );
  }

  [Fact]
  public void when_a_double_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const int body = 99016;
    var person = new Person { Address = new Address { ZipCode = new ZipCode { Body = body } } };
    int value = person.ValueOrDefault( p => p.Address.ZipCode.Body );
    Assert.Equal( body, value );
  }

  [Fact]
  public void when_a_triple_nested_accessed_propertys_parent_is_null_then_null_is_returned()
  {
    var person = new Person { Address = null };
    string value = person.ValueOrDefault( p => p.Address.ZipCode.Zone.Name );
    Assert.Equal( (object) null, value );
  }

  [Fact]
  public void when_a_triple_nested_accessed_property_is_not_null_then_its_value_is_returned()
  {
    const string name = "Yay";
    var person = new Person
                   { Address = new Address { ZipCode = new ZipCode { Zone = new Zone { Name = name } } } };
    string value = person.ValueOrDefault( p => p.Address.ZipCode.Zone.Name );
    Assert.Equal( name, value );
  }
}
```

