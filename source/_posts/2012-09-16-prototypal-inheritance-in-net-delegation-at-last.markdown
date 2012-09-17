---
layout: post
title: "Prototypal Inheritance in .NET: Delegation at Last"
date: 2012-09-16 17:19
comments: true
categories: 
published: false
---
In .NET 4.0, we have access to the [Dynamic Language Runtime (DLR)][] giving us [dynamic dispatch][] via the [IDynamicMetaObjectProvider][] interface coupled with the [dynamic][] keyword. When a variable is declared as ```dynamic``` and it implements the ```IDynamicMetaObjectProvider``` interface, we are given the opportunity to control the delegation of all calls on that object by returning a ```DynamicMetaObject``` containing an expression which will be evaluated by the runtime.

The ```DynamicObject``` and ```ExpandoObject``` classes both implement ```IDynamicMetaObjectProvider```, but their implementations are explicitly implemented and the nested classes used to return the ```DynamicMetaObject``` are private and sealed. I understand that the classes may not have been tested enough to make them inheritable, but not having access to these classes really hurts our ability to easily modify the behavior of the underlying ```DynamicMetaObject```. The key word here is easily; implementing the needed expression building is a great deal of work and the internal workings of the Microsoft implementations leverage many internal framework calls.

A key thing to consider is that we don't want to replicate classical inheritance; instead, we are going to focus on prototypal inheritance that mostly replicates JavaScript's prototypal inheritance. Rather than trying to replicate all of the hard work that the DRL team put into writing their implementation, we can add our own implementation on top of theirs. It is simple to hook in, but we need to save off a method to access the base ```DynamicMetaObject``` implementation. This will allow us to attempt to interpret the expression on the object itself or pass it along.

``` csharp
public interface IPrototypalMetaObjectProvider : IDynamicMetaObjectProvider {
    object Prototype { get; }
    DynamicMetaObject GetBaseMetaObject( Expression parameter );
}

public class DelegatingPrototype : DynamicObject, IPrototypalMetaObjectProvider {
    public DelegatingPrototype() : this( null ) { }

    public DelegatingPrototype( object prototype ) {
        Prototype = prototype;
    }

    public virtual object Prototype { get; set; }

    public override DynamicMetaObject GetMetaObject( Expression parameter ) {
        if ( Prototype == null ) {
            return GetBaseMetaObject( parameter );
        }
        return new PrototypalMetaObject( parameter, this, Prototype );
    }

    public virtual DynamicMetaObject GetBaseMetaObject( Expression parameter ) {
        return base.GetMetaObject( parameter );
    }
}
```

This small amount of code just sets the hook. Now we need set up the delegation expression. 

{% blockquote L. Peter Deutsch, %}
To iterate is human, to recurse divine
{% endblockquote %}

To set up the prototypal hierarchy, we are going to need to do a lot of recursion. Unfortunately, it is well hidden (I'll explain shortly). When a call is made on the root object, we are given the expression being interpreted. Using the ```IDynamicMetaObjectProvider``` overload, we will hand off the expression to the ```PrototypalMetaObject``` to construct delegation expression (or the ```DynamicObject``` implementation if our prototype is ```null``` thus trying to interpret the expression on the current object). We want to make a preorder traversal of the prototype hierarchy; at each step, the current object's evaluation will take precedence over its prototype tree. Consider the following prototypal class hierarchy:

{% graphviz %}
digraph G {
  bgcolor="transparent";
  compound=true;
  subgraph Square {
    Square -> s1;
    s1 [label="self"];
    Square -> Quadrilateral [label="prototype"];
      subgraph Quadrilateral {
      Quadrilateral -> s2; 
      s2 [label="self"];
      Quadrilateral -> Polygon [label="prototype"];
      subgraph Polygon {
        Polygon -> s3;
        s3 [label="self"];
        Polygon -> null [label="prototype"];
      }
    }
  }
}
{% endgraphviz %}

Since we never know which level of the hierarchy will be handling the expression, we need to build an expression for the entire tree every time. We want to get the ```DynamicMetaObject``` representing the current object's tree first. Once done, we get the  ```DynamicMetaObject``` for evaluating the expression on the current instance. With these two, we can create a new  ```DynamicMetaObject``` which try to bind the expression to the current instance first, and then fallback to the prototype. At the root level, the prototype ```DynamicMetaObject``` contains the same fallback for the next two layers.

```Square fallback => Quadrilateral fallback => Polygon```

There is another caveat that we need to address. When we try to invoke and expression on an object, the expression is bound to that type. When accessing the prototype, if we don't do anything, the system will throw a binding exception because the matching object won't match the ```DynamicMetaObject```'s type restrictions. To fix this, we need to relax the type restrictions for each prototype.

``` csharp
public class PrototypalMetaObject : DynamicMetaObject {
    private readonly DynamicMetaObject _baseMetaObject;
    private readonly DynamicMetaObject _metaObject;
    private readonly IPrototypalMetaObjectProvider _prototypalObject;
    private readonly object _prototype;

    public PrototypalMetaObject( Expression expression, IPrototypalMetaObjectProvider value, object prototype )
            : base( expression, BindingRestrictions.Empty, value ) {
        _prototypalObject = value;
        _prototype = prototype;
        _metaObject = CreatePrototypeMetaObject();
        _baseMetaObject = CreateBaseMetaObject();
    }

    protected virtual DynamicMetaObject AddTypeRestrictions( DynamicMetaObject result ) {
        BindingRestrictions typeRestrictions = GetTypeRestriction().Merge( result.Restrictions );
        return new DynamicMetaObject( result.Expression, typeRestrictions, _metaObject.Value );
    }

    protected virtual DynamicMetaObject CreatePrototypeMetaObject() {
        Expression castExpression = GetLimitedSelf();
        MemberExpression memberExpression = Expression.Property( castExpression, "Prototype" );
        return Create( _prototype, memberExpression );
    }

    protected virtual BindingRestrictions GetTypeRestriction() {
        if ( Value == null && HasValue ) {
            return BindingRestrictions.GetInstanceRestriction( Expression, null );
        }
        return BindingRestrictions.GetTypeRestriction( Expression, LimitType );
    }

    protected virtual DynamicMetaObject CreateBaseMetaObject() {
        return _prototypalObject.GetBaseMetaObject( Expression );
    }

    protected Expression GetLimitedSelf() {
        return AreEquivalent( Expression.Type, LimitType ) ? Expression : Expression.Convert( Expression, LimitType );
    }

    protected bool AreEquivalent( Type lhs, Type rhs ) {
        return lhs == rhs || lhs.IsEquivalentTo( rhs );
    }
//...
    public override DynamicMetaObject BindGetMember( GetMemberBinder binder ) {
        DynamicMetaObject errorSuggestion = AddTypeRestrictions( _metaObject.BindGetMember( binder ) );
        return binder.FallbackGetMember( _baseMetaObject, errorSuggestion );
    }
//...
    public override DynamicMetaObject BindInvokeMember( InvokeMemberBinder binder, DynamicMetaObject[] args ) {
        DynamicMetaObject errorSuggestion = AddTypeRestrictions( _metaObject.BindInvokeMember( binder, args ) );
        return binder.FallbackInvokeMember( _baseMetaObject, args, errorSuggestion );
    }
//...
    public override DynamicMetaObject BindSetMember( SetMemberBinder binder, DynamicMetaObject value ) {
        DynamicMetaObject errorSuggestion = AddTypeRestrictions( _metaObject.BindSetMember( binder, value ) );
        return binder.FallbackSetMember( _baseMetaObject, value, errorSuggestion );
    }
//...
}
```

You may be thinking, ok, this is cool, but what use it is it? What is the use case? Primarily, it is fun. Secondly, it sets the foundation for .NET [mixins][].

If you want to see more or play around with the code, you can find full implementations in the [Archetype][] project.

[Dynamic Language Runtime (DLR)]: http://en.wikipedia.org/wiki/Dynamic_Language_Runtime
[dynamic dispatch]: http://en.wikipedia.org/wiki/Dynamic_dispatch
[IDynamicMetaObjectProvider]: http://msdn.microsoft.com/en-us/library/system.dynamic.idynamicmetaobjectprovider(v=vs.100).aspx
[dynamic]: http://msdn.microsoft.com/en-us/library/dd264741(v=vs.100).aspx
[mixins]: http://en.wikipedia.org/wiki/Mixin
[Archetype]: https://github.com/idavis/Archetype