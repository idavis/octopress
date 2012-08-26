---
layout: post
title: "Project Euler: Problem 7 - The Nth Prime"
date: 2012-08-25 17:45
comments: true
categories: [powerhell, projecteuler]
published: false
---
``` ps1 Prototypal
function New-PrimeGenerator {
  $prototype = New-Prototype
  $prototype | Update-TypeName
  $prototype | Add-Property Primes @(2,3,5)
  $prototype | Add-Property Cache @{2=$true; 3=$true; 4=$false; 5=$true}
  $prototype | Add-ScriptProperty MaxPrime {$this.Primes | select -last 1}
  $prototype | Add-Property Bound 5
  $prototype | Add-Property BoundIncrement 1000
  $prototype | Add-Property LastExpansionPrimeCount 3
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
      for ($i=$this.RoundToNextMultiple($oldBound,$_); $i -le $limit; $i += $_ ) { if(!$cache.ContainsKey($i)) { $cache[$i] = $false } }
    }
    $this.LastExpansionPrimeCount = $this.Primes.Length
  }
  $prototype
}
 
$generator = New-PrimeGenerator
(Measure-Command {$generator.FindNthPrime(10001)}).TotalSeconds
33.1381042s
(Measure-Command {$generator.FindNthPrime(10001)}).TotalSeconds
0.0003987s
$generator.FindNthPrime(10001)
# The Answer
104743
```