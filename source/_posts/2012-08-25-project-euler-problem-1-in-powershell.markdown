---
layout: post
title: "Project Euler: Problem 1 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---

{% blockquote Project Euler http://projecteuler.net/problem=1 Problem 1 %}
If we list all the natural numbers below 10 that are multiples of 3 or 5, we get 3, 5, 6 and 9. The sum of these multiples is 23.

Find the sum of all the multiples of 3 or 5 below 1000.
{% endblockquote %}

The first problem is rather easy. We just need to generate a sequence, filter it, and them sum the results. Calculating sums in PowerShell is syntactically awkward as you can see in the different examples.

One item that will probably stand out as odd is the `| % -begin {} -process {} -end {}` in which I am effectively creating a stateful function in the pipe.

``` ps1
function Solve-Problem1 {
  # using Measure-Object
  (1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0} | Measure-Object -Sum).Sum

  # using begin, process, end blocks
  1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % -begin {$sum=0} -process {$sum+=$_} -end {$sum}

  # cleaning up a bit
  1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % {$sum = 0} {$sum += $_} {$sum}
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem1}).TotalSeconds
Write-Host "Solution: " (Solve-Problem1)

Elapsed Time (s):  0.0606135
Solution:  233168
```
