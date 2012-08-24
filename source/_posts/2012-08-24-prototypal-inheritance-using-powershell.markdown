---
layout: post
title: "Prototypal Inheritance Using PowerShell"
date: 2012-08-24 10:15
comments: true
categories: [powershell, patterns]
published: false
---
## Prototype-based programming
{% blockquote Wikipedia http://en.wikipedia.org/wiki/Prototype-based_programming Prototype-based programming %}
Prototype-based programming is a style of object-oriented programming in which classes are not present, and behavior reuse (known as inheritance in class-based languages) is performed via a process of cloning existing objects that serve as prototypes. This model can also be known as classless, prototype-oriented or instance-based programming.
{% endblockquote %}



## Issue of delegation
{% blockquote Wikipedia http://en.wikipedia.org/wiki/Prototype-based_programming Prototype-based programming %}
In prototype-based languages that use delegation, the language runtime is capable of dispatching the correct method or finding the right piece of data [...]
{% endblockquote %}

PowerShell, even the new DRL based v3, does not support true [dynamic dispatch][]. If we could have true dynamic dispatch, we could mimic JavaScript's prototypal inheritance with each object having an actual prototype to which we could delegate messages and support overriding.







  [dynamic dispatch]: http://en.wikipedia.org/wiki/Dynamic_dispatch