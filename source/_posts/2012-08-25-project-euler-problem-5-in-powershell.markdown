---
layout: post
title: "Project Euler: Problem 5 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---
{% blockquote Project Euler http://projecteuler.net/problem=5 Problem 5 %}
2520 is the smallest number that can be divided by each of the numbers from 1 to 10 without any remainder.

What is the smallest positive number that is evenly divisible by all of the numbers from 1 to 20?
{% endblockquote %}

The next problem is just folding two values from an array and replacing them with their LCM, then run again and again until only one item is left in the collection (and you now have the LCM of all entries).

``` ps1 What is the smallest positive number that is evenly divisible by all of the numbers from 1 to 20?
function Get-Gcd {
  param($lhs, $rhs)
  if ($lhs -eq $rhs) { return $rhs }
  if ($lhs -gt $rhs) { $a,$b = $lhs,$rhs }
  else { $a,$b = $rhs,$lhs }
  while ($a % $b -ne 0) {
    $tmp = $a % $b
    $a,$b = $b,$tmp
  }
  return $b
}

function Get-Lcm {
  param($lhs, $rhs)
  [long][Math]::Abs($lhs * $rhs) / (Get-Gcd $lhs $rhs)
}

function Get-LcmOfGroup {
  param([int[]]$values)
  if($values.Length -lt 2) { return -1 }
  $lhs = $values[0]
  $rhs = $values[1]
  $lcm = Get-Lcm $lhs $rhs
  $values = @($lcm) + $values[2..$values.Length]
  if($values.Length -eq 1) { return $lcm }
  else { Get-LcmOfGroup $values }
}

Get-LcmOfGroup @(1..20)

# The Answer
232792560
```
