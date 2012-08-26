---
layout: post
title: "Project Euler: Problem 3 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---

Generating prime factors is another simple algorithm. We divide off all of the 2's, then we take off the 3's, but from there we skip two instead of one as we already pulled out all of the possible even factors.

``` ps1 What is the largest prime factor of the number 600851475143
function Get-PrimeFactors {
  param([long]$number)
  while($number % 2 -eq 0) {
     2
     $number /= 2
  }

  for($i = 3; $i -le [Math]::Sqrt($number); $i += 2) {
     while($number % $i -eq 0) {
        $number /= $i
        $i
     }
  }
  $number
}

Get-PrimeFactors 600851475143 | select -last 1

# The answer
6857
```
