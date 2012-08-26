---
layout: post
title: "Project Euler: Problem 7 in Prototypal PowerShell"
date: 2012-08-25 17:45
comments: true
categories: [powerhell, projecteuler]
published: false
---
This post leverages my OSS project [Prototye.ps][] to support prototypal object creation. If you haven't read the introduction posts [Part 1][] and [Part 2][], I would recommend reading them as I build off of their functionality and theory. If you don't care how it works, read on my friend.

{% blockquote Project Euler http://projecteuler.net/problem=7 Problem 7 %}
By listing the first six prime numbers: 2, 3, 5, 7, 11, and 13, we can see that the 6th prime is 13.

What is the 10 001st prime number?
{% endblockquote %}

``` ps1 Solving Using a Prototypal Object
function New-PrimeGenerator {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype | Add-Property Primes @(2,3,5)
  $prototype | Add-Property Cache @{2=$true; 3=$true; 4=$false; 5=$true}
  $prototype | Add-Property Bound 5
  $prototype | Add-Property BoundIncrement 1000
  $prototype | Add-Property LastExpansionPrimeCount 3
  $prototype | Add-ScriptProperty MaxPrime {$this.Primes | select -last 1}
  $prototype | Add-Function IsPrime { 
    param($value)
    while ($this.MaxPrime -lt $value) { $this.Expand() }
    return $this.Cache[$value] -ne $null
  }
  $prototype | Add-Function FindNthPrime { 
    param($value)
    while ($this.Primes.Length -lt $value) { $this.Expand() }
    return $this.Primes[$value-1]
  }
  $prototype | Add-Function RoundToNextMultiple {
    param($base, $multiple)
    $remainder = $base % $multiple;
    if($remainder -eq 0) { $base }
    else { $base + $multiple - $remainder }
  }
  $prototype | Add-Function Expand {
    $oldBound = $this.Bound
    if($this.LastExpansionPrimeCount -le $this.Primes.Length) {
      $this.Bound += $this.BoundIncrement
    }
    $limit= $this.Bound
    $cache = $this.Cache
    $this.Primes | % {
      for ($i=$this.RoundToNextMultiple($oldBound,$_); $i -le $limit; $i += $_ ) { 
        if(!$cache.ContainsKey($i)) { $cache[$i] = $false }
      }
    }
    ($oldBound+1)..$limit | ? { $cache[$_] -eq $null } | % {
      $cache[$_] = $true
      $this.Primes += @($_)
      for ($i=$this.RoundToNextMultiple($oldBound,$_); $i -le $limit; $i += $_ ) {
        if(!$cache.ContainsKey($i)) { $cache[$i] = $false }
      }
    }
    $this.LastExpansionPrimeCount = $this.Primes.Length
  }
  $prototype
}
 
function Solve-Problem7 {
  $generator = New-PrimeGenerator
  Write-Host "Elapsed Time (s): " (Measure-Command {$generator.FindNthPrime(10001)}).TotalSeconds
  Write-Host "Elapsed Time (s): " (Measure-Command {$generator.FindNthPrime(10001)}).TotalSeconds
  Write-Host "Solution: " ($generator.FindNthPrime(10001))
}

Solve-Problem7

Elapsed Time (s):  33.1381042
Elapsed Time (s):   0.0003987
Solution:  104743
```


  [Prototye.ps]: https://github.com/idavis/prototype.ps
  [Part 1]: /2012/08/prototypal-inheritance-using-powershell
  [Part 2]: /2012/08/prototypal-inheritance-using-powershell-part-two-scriptproperties