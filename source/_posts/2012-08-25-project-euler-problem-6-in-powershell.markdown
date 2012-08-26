---
layout: post
title: "Project Euler: Problem 6 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---

``` ps1 Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.
$sum = (1..100 | Measure-Object -Sum).Sum
$sumOfSquares = (1..100 | % {$_*$_} | Measure-Object -Sum).Sum
$sum * $sum - $sumOfSquares

# The Answer
25164150
```
