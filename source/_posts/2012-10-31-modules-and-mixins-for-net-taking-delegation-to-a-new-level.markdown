---
layout: post
title: "Modules and Mixins for .NET: Taking Delegation to a New Level"
date: 2012-10-31 20:58
comments: true
categories: [dynamic, c#, dlr]
---
In the last article on [prototypal inheritance][], I showed how to construct a delegation chain and simulate JavaScript's prototypal inheritance for .NET languages. If you haven't looked at that post, I highly recommend reading it through as I am going to skip over a lot of the theory I covered previously. Today I am taking delegation in .NET a step further. If you aren't familiar with mixins and ruby modules, I would also recommend looking into them as what I am showing here is directly inspired by them.

When looking at prototypal inheritance for .NET, we had a very simple delegation chain. With module support we need to ammend how we evaluate and interpret expressions being called on our objects. Does the current object support the operation? Can it respond? If not, we need to loop through the prototypes (modules/mixins) that have been attached to this instance. There is a catch. We need to process them last to first and at each level we need to start the evaluation all over again. Why last first?

When we declare a module/mixin for our class, it is taking precededence over the modules that have already been imported. This mimics the way that Ruby works. Defining a property or method that already exists redefines that member.

Revisiting the ```DelegatingPrototype```, I have added a new backing list to replace the single prototype from the earlier version. All existing tests and code continue to work just fine with these minimal changes. The big change is the call to the ```PrototypalMetaObject``` ```ctor```. We are passing a collection of prototypes/modules which will be evaluated by the ```PrototypalMetaObject``` which may contain many more sets of modules and delegation chains.
``` csharp
public class DelegatingPrototype : DynamicObject, IPrototypalMetaObjectProvider {
    private readonly List<object> _Prototypes = new List<object>();

    public DelegatingPrototype() : this(null) { }

    public DelegatingPrototype(object prototype) {
        _Prototypes.Add(prototype);
    }

    #region IPrototypalMetaObjectProvider Members

    public virtual object Prototype {
        get { return _Prototypes.FirstOrDefault(); }
        set { _Prototypes[0] = value; }
    }

    public virtual IList<object> Prototypes {
        get { return _Prototypes; }
    }

    public override DynamicMetaObject GetMetaObject(Expression parameter) {
        if (Prototypes.Count == 0 || Prototype == null) {
            return GetBaseMetaObject(parameter);
        }
        return new PrototypalMetaObject(parameter, this, _Prototypes);
    }

    public virtual DynamicMetaObject GetBaseMetaObject(Expression parameter) {
        return base.GetMetaObject(parameter);
    }

    #endregion
}
```

Just like last time, I have removed all binding calls with the exception of the ```BindGetMember```. Now, instead of executing a preorder traversal of the prototype chains, we are processing the routing all binding calls to a new method ```ApplyBinding``` which isolates all of the resolution logic to a single method. We have an additional constraint this time in that we are binding the expression to delegation chains that may fail (whereas before we had only a single chain that could fail). When this binding failure occurs, an expression with a ```NodeType``` of ```ExpressionType.Throw``` is returned and we need to ignore these failed chains. I am also able to leverage ```Func<,>```, ```Func<,,>```, and closures to capture the binding calls and context arguments which makes every override an easy call to ```ApplyBinding```. You can take a look at the [full file][] to see how simply everything is laid out.

``` csharp
public class PrototypalMetaObject : DynamicMetaObject
{
    private readonly DynamicMetaObject _baseMetaObject;
    private readonly IPrototypalMetaObjectProvider _prototypalObject;
    private readonly IList<object> _prototypes;

    public PrototypalMetaObject(Expression expression, IPrototypalMetaObjectProvider value, object prototype)
        : this(expression, value, new[] {prototype}) {
    }

    public PrototypalMetaObject(Expression expression, IPrototypalMetaObjectProvider value, IList<object> prototypes)
        : base(expression, BindingRestrictions.Empty, value) {
        _prototypalObject = value;
        _prototypes = prototypes;
        _baseMetaObject = CreateBaseMetaObject();
    }

    protected virtual DynamicMetaObject AddTypeRestrictions(DynamicMetaObject result, DynamicMetaObject value) {
        BindingRestrictions typeRestrictions = GetTypeRestriction().Merge(result.Restrictions);
        var metaObject = new DynamicMetaObject(result.Expression, typeRestrictions, value.Value);
        return metaObject;
    }

    protected virtual BindingRestrictions GetTypeRestriction() {
        if (Value == null && HasValue) {
            return BindingRestrictions.GetInstanceRestriction(Expression, null);
        }
        return BindingRestrictions.GetTypeRestriction(Expression, LimitType);
    }

    protected virtual DynamicMetaObject CreateBaseMetaObject() {
        DynamicMetaObject baseMetaObject = _prototypalObject.GetBaseMetaObject(Expression);
        return baseMetaObject;
    }

    protected Expression GetLimitedSelf() {
        return AreEquivalent(Expression.Type, LimitType)
                   ? Expression
                   : Expression.Convert(Expression, LimitType);
    }

    protected bool AreEquivalent(Type t1, Type t2) {
        return t1 == t2 || t1.IsEquivalentTo(t2);
    }
	
    public override DynamicMetaObject BindGetMember(GetMemberBinder binder) {
        return ApplyBinding(meta => meta.BindGetMember(binder), binder.FallbackGetMember);
    }
		  
    protected virtual DynamicMetaObject ApplyBinding(Func<DynamicMetaObject, DynamicMetaObject> bindTarget,
                                                     Func<DynamicMetaObject, DynamicMetaObject, DynamicMetaObject> bindFallback) {
        DynamicMetaObject errorSuggestion = null;
        for (int i = _prototypes.Count - 1; i >= 0; i--) {
            object prototype = _prototypes[i];

            DynamicMetaObject prototypeMetaObject = Create(prototype, Expression.Constant(prototype));
            if (errorSuggestion == null) {
                errorSuggestion = AddTypeRestrictions(bindTarget(prototypeMetaObject), prototypeMetaObject);
                if (errorSuggestion.Expression.NodeType == ExpressionType.Throw) {
                    errorSuggestion = null;
                }
            }
            else
            {
                DynamicMetaObject newValue = AddTypeRestrictions(bindTarget(prototypeMetaObject), prototypeMetaObject);
                if (newValue.Expression.NodeType == ExpressionType.Throw) {
                    continue;
				}
				
                errorSuggestion = bindFallback(newValue, errorSuggestion);
            }
        }

        return bindFallback(_baseMetaObject, errorSuggestion);
    }
}
```

With these two classes now in place, it is time to start having some fun. Invariably, when talking about module support in .NET, one of the most common requests you will here people talk about is to have a ```INotifyPropertyChanged``` module.




  [prototypal inheritance]: http://codepyre.com/2012/10/prototypal-inheritance-in-net-delegation-at-last/
  [full file]: https://github.com/idavis/Archetype/blob/8a55daf4e3484e4dca141ec676f83db8dd76e35a/src/Archetype/PrototypalMetaObject.cs