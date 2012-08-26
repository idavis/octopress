---
layout: post
title: "Project Euler: Problem 4 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---

``` ps1 Find the largest palindrome made from the product of two 3-digit numbers.
(100..999 | % {
  $lhs = $_
  100..999 | % {
    $rhs = $_
    $product = $lhs * $rhs
    $value = $product.ToString()
    if($value -eq ($value[$value.Length..0] -join "")) {$product}
  }
} | Measure-Object -Maximum).Maximum

# The answer
906609
```
