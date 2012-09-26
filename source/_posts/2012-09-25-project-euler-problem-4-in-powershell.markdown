---
layout: post
title: "Project Euler: Problem 4 in PowerShell"
date: 2012-09-25 20:00:58
comments: true
categories: [powershell, projecteuler]
---
{% blockquote Project Euler http://projecteuler.net/problem=4 Problem 4 %}
A palindromic number reads the same both ways. The largest palindrome made from the product of two 2-digit numbers is 9009 = 91*99.

Find the largest palindrome made from the product of two 3-digit numbers.
{% endblockquote %}

``` ps1
function Solve-Problem4 {
  (100..999 | % {
    $lhs = $_
    100..999 | % {
      $rhs = $_
      $product = $lhs * $rhs
      $value = $product.ToString()
      if($value -eq ($value[$value.Length..0] -join "")) {$product}
    }
  } | Measure-Object -Maximum).Maximum
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem4}).TotalSeconds
Write-Host "Solution: " (Solve-Problem4)

Elapsed Time (s):  87.4686333
Solution:  906609
```
