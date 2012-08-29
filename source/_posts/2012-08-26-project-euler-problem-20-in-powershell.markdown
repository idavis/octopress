---
layout: post
title: "Project Euler: Problem 20 in PowerShell"
date: 2012-08-26 13:12
comments: true
categories: [powershell, projecteuler]
published: false
---
{% blockquote Project Euler http://projecteuler.net/problem=20 Problem 20 %}
n! means n  (n  1)  ...  3  2  1

For example, 10! = 10  9  ...  3  2  1 = 3628800,
and the sum of the digits in the number 10! is 3 + 6 + 2 + 8 + 8 + 0 + 0 = 27.

Find the sum of the digits in the number 100!
{% endblockquote %}

``` ps1
filter Evaluate-String {
  $_.ToString().ToCharArray() | % { [int]::Parse($_) } | % {$total = 0} {$total += $_} {$total}
}
function Solve-Problem20 {
  100..1 | % {$value = New-Object Numerics.BigInteger 1} {$value *= $_} {$value} | Evaluate-String
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem20}).TotalSeconds
Write-Host "Solution: " (Solve-Problem20)

Elapsed Time (s):  0.0191042
Solution:  648
```