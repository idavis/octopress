---
layout: post
title: "Project Euler: Problem 2 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---

The [Fibonacci sequence][] is much more fun. You can get close to this implementation with `yield return` in C#, but the multiple assignment trick has yet to come to C# :(.

``` ps1 Find the sum of the even-valued terms in the Fibonacci sequence whose values do not exceed four million
function Get-FibonacciSequence {
  param([int]$max)
  0
  1
  for($i = $j = 1; $i -lt $max) {
    $i
    $i,$j = ($i + $j),$i
   }
}

filter IsEven { if($_ % 2 -eq 0) { $_ } }

((Get-FibonacciSequence 4000000) | IsEven | Measure-Object -Sum).Sum

# The answer
4613732 
```

  [Fibonacci sequence]: http://en.wikipedia.org/wiki/Fibonacci_number