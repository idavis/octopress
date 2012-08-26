---
layout: post
title: "Project Euler: Problem 6 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---
{% blockquote Project Euler http://projecteuler.net/problem=6 Problem 6 %}
The sum of the squares of the first ten natural numbers is,

1^2 + 2^2 + ... + 10^2 = 385
The square of the sum of the first ten natural numbers is,

(1 + 2 + ... + 10)^2 = 55^2 = 3025
Hence the difference between the sum of the squares of the first ten natural numbers and the square of the sum is 3025  385 = 2640.

Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.
{% endblockquote %}

``` ps1 Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.
$sum = (1..100 | Measure-Object -Sum).Sum
$sumOfSquares = (1..100 | % {$_*$_} | Measure-Object -Sum).Sum
$sum * $sum - $sumOfSquares

# The Answer
25164150
```
