---
layout: post
title: "Project Euler: Problem 10 in PowerShell"
date: 2012-10-02 13:20
comments: true
categories: [powershell, projecteuler]
---
{% blockquote Project Euler http://projecteuler.net/problem=10 Problem 10 %}
The sum of the primes below 10 is 2 + 3 + 5 + 7 = 17.

Find the sum of all the primes below two million.
{% endblockquote %}

This problem is very simple as I can reuse the parasitic prime generation from my solution to [Problem 7][] which also requires [Prototype.ps][].

``` ps1
function New-PrimeFinder {
  $prototype = New-PrimeGenerator
  $prototype | Add-Function FindPrimesLessThan { 
    param($value)
    if($this.Bound -lt $value) { 
      $finder.BoundIncrement = $value - $this.Bound
      $this.Expand()
    }
    return $this.Primes | ? { $_ -lt $value }
  }
  $prototype
}

function Solve-Problem10 {
  param($value = 2000000)
  $finder = New-PrimeFinder
  $finder.FindPrimesLessThan($value) | % {[long]$sum = 0} {$sum+=$_} {$sum}
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem10}).TotalSeconds
Write-Host "Solution: " (Solve-Problem10)

Elapsed Time (s):  1999.6739304
Solution:  142913828922
```

  [Problem 7]: /2012/08/project-euler-problem-7-in-parasitic-powershell
  [Prototye.ps]: https://github.com/idavis/prototype.ps
