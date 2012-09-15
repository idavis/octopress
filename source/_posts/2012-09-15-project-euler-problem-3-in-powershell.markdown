---
layout: post
title: "Project Euler: Problem 3 in PowerShell"
date: 2012-09-15 20:00:00
comments: true
categories: [powershell, projecteuler]
---
{% blockquote Project Euler http://projecteuler.net/problem=3 Problem 3 %}
The prime factors of 13195 are 5, 7, 13 and 29.

What is the largest prime factor of the number 600851475143 ?
{% endblockquote %}

Generating prime factors is another simple algorithm. We divide off all of the 2's, then we take off the 3's, but from there we skip two instead of one as we already pulled out all of the possible even factors.

``` ps1
function Get-PrimeFactors {
  param([long]$number)
  while($number % 2 -eq 0) {
     2
     $number /= 2
  }
  $bound = [Math]::Sqrt($number)
  for($i = 3; $i -le $bound; $i += 2) {
     while($number % $i -eq 0) {
        $number /= $i
        $i
     }
  }
  $number
}

function Solve-Problem3 {
  Get-PrimeFactors 600851475143 | select -last 1  
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem3}).TotalSeconds
Write-Host "Solution: " (Solve-Problem3)

Elapsed Time (s):  0.0477742
Solution:  6857
```
