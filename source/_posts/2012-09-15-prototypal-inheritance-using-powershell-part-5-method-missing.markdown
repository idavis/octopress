---
layout: post
title: "Prototypal Inheritance Using PowerShell (Part 5): Method Missing"
date: 2012-09-15 16:19
comments: true
categories: 
---
If you didn't read [Part 1][] and [Part 2][], I would recommend reading them as I build off of their functionality and theory.

So far, we still don't have real prototypal inheritance, but instead have added an API for parasitic inheritance:

{% blockquote Douglas Crockford http://www.crockford.com/javascript/inheritance.html Parasitic Inheritance %}
Instead of inheriting from Parenizor, we write a constructor that calls the Parenizor constructor, passing off the result as its own. And instead of adding public methods, the constructor adds privileged methods.
``` js
function ZParenizor2(value) {
    var that = new Parenizor(value);
    that.toString = function () {
        if (this.getValue()) {
            return this.uber('toString');
        }
        return "-0-"
    };
    return that;
}
```
Classical inheritance is about the is-a relationship, and parasitic inheritance is about the was-a-but-now's-a relationship.
{% endblockquote %}

Creating full prototypal inheritance is going to require a lot of work, but we can take a first step by adding single level delegation via ```DynamicObject``` in PowerShell 3.0 (.NET 4). When an object does not contain the member being invoked, ```DynamicObject```'s implementation of ```IDynamicMetaObjectProvider``` will fallback by calling the ```TryXyz``` method of its inheritor. We can override the base implementation of the ```TryXyz``` methods giving us a last chance to handle the missing expression invocation.

Using ```PSObject```, we can attach methods, properties, and variables. By creating a new ```PrototypalObject``` class which we wrap in a ```PSObject```, we get a single level of prototypal inheritance with method missing. If we just look at get/set member and method invocation, our ```PrototypalObject``` might look like this:

``` csharp
namespace Archetype
{
    using System;
    using System.Dynamic;

    public delegate bool TryGetMemberMissing( GetMemberBinder binder, out object result );
    public delegate bool TryInvokeMemberMissing( InvokeMemberBinder binder, object[] args, out object result );
    public delegate bool TrySetMemberMissing( SetMemberBinder binder, object value );
// ...
    
    public class PrototypalObject : DynamicObject
    {
        public PrototypalObject() : this(null) { }

        public PrototypalObject(DynamicObject prototype) {
            Prototype = prototype;
        }

        public DynamicObject Prototype { get; set; }
        public virtual TryGetMemberMissing TryGetMemberMissing { get; set; }
        public virtual TryInvokeMemberMissing TryInvokeMemberMissing { get; set; }
        public virtual TrySetMemberMissing TrySetMemberMissing { get; set; }
// ...
        public override bool TryGetMember( GetMemberBinder binder, out object result ) {
            if(Prototype != null) {
                if(Prototype.TryGetMember(binder, out result)) {
                    return true;
                }
            }
            if ( TryGetMemberMissing == null ) {
                result = null;
                return false;
            }
            return TryGetMemberMissing( binder, out result );
        }

        public override bool TryInvokeMember( InvokeMemberBinder binder, object[] args, out object result ) {
            if(Prototype != null) {
                if(Prototype.TryInvokeMember(binder, args, out result)) {
                    return true;
                }
            }
            if ( TryInvokeMemberMissing == null ) {
                result = null;
                return false;
            }
            return TryInvokeMemberMissing( binder, args, out result );
        }

        public override bool TrySetMember( SetMemberBinder binder, object value ) {
            if(Prototype != null) {
                if(Prototype.TrySetMember(binder, value)) {
                    return true;
                }
            }
            if ( TrySetMemberMissing == null ) {
                return false;
            }
            return TrySetMemberMissing( binder, value );
        }
// ...
    }
}
```

By setting the ```TryXyzMissing``` members, we can hook into the missing method/invocation/indexing of our object. Now we just need to add the ability to load our prototype into the existing API.

``` ps1
$here = Split-Path -Parent $MyInvocation.MyCommand.Path

function New-Prototype {
  param(
    $baseObject = (new-object object)
  )
  process {
    $prototype = $null
    if($PSVersionTable.CLRVersion.Major -lt 4) {
	  # we have to fall back to parasitic inheritance
      $prototype = [PSObject]::AsPSObject($baseObject)
      $prototype.PSObject.TypeNames.Insert(0,"Prototype")
    } else {
      Import-PrototypalObject # verifies that Archetype.PrototypalObject is loaded
      $pso = [PSObject]::AsPSObject($baseObject)
      $dispatcher = (New-Object Archetype.PrototypalObject -ArgumentList $pso)
      $prototype = [PSObject]::AsPSObject($dispatcher)
    }
    
    $prototype | Add-StaticInstance
    $prototype
  }
}

function Import-PrototypalObject {
  param()
  if(@(try{[Archetype.PrototypalObject]}catch{}).Length -eq 0 ) {
    Add-Type -Path "$here\Prototype.cs" -ReferencedAssemblies @("System.Core", "Microsoft.CSharp")
  }
}
````

We now have a very simple version of method missing for PowerShell and C#. The next post discusses true prototypal inheritance for C# and as a side effect, PowerShell.

[Part 1]: /2012/08/prototypal-inheritance-using-powershell
[Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties

