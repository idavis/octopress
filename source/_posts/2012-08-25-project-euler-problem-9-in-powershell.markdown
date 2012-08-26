---
layout: post
title: "Project Euler: Problem 9 in PowerShell"
date: 2012-08-25 20:15
comments: true
categories: [powerhell, projecteuler]
published: false
---
{% blockquote Project Euler http://projecteuler.net/problem=9 Problem 9 %}
A Pythagorean triplet is a set of three natural numbers, a < b < c, for which,
   a^2 + b^2 = c^2
For example, 32 + 42 = 9 + 16 = 25 = 52.

There exists exactly one Pythagorean triplet for which a + b + c = 1000.
Find the product abc.
{% endblockquote %}

``` ps1
function Solve-Problem9 {
  $sum = 1000
  $found = $false
  for ($a = 1; $a -lt $sum / 3; $a++) {
    for ($b = $a; $b -lt $sum / 2; $b++) {
      $c = $sum - $a - $b
 
      if ($a * $a + $b * $b -eq $c * $c) {
        $found = $true
        break
      }
    }
 
    if ($found) { break }
  }

  Write-Host "The Pythagorean triplet is a = $a, b = $b, c = $c"
  Write-Host "The product is $($a*$b*$c)" 
}

Solve-Problem9
The Pythagorean triplet is a = 200, b = 375, c = 425
The product is 31875000
```
