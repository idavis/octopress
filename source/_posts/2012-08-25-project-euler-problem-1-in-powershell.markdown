---
layout: post
title: "Project Euler: Problem 1 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: [powerhell, projecteuler]
published: false
---
The first problem is rather easy. We just need to generate a sequence, filter it, and them sum the results. Calculating sums in PowerShell is syntactically awkward as you can see in the different examples.

``` ps1 Add all the natural numbers below one thousand that are multiples of 3 or 5.
# using Measure-Object
(1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0} | Measure-Object -Sum).Sum

# using begin, process, end blocks
1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % -begin {$sum=0 } -process {$sum+=$_} -end {$sum}

# cleaning up a bit
1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % {$sum = 0} {$sum += $_} {$sum}

# The answer
233168 
```
