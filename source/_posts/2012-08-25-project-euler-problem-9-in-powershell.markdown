---
layout: post
title: "Project Euler: Problem 9 in PowerShell"
date: 2012-08-25 20:15
comments: true
categories: [powerhell, projecteuler]
published: false
---

``` ps1
$a,$b, $c = 0
$sum = 1000
$found = $false
for ($a = 1; $a -lt $sum / 3; $a++) {
    for ($b = $a; $b -lt $sum / 2; $b++) {
        $c = $sum - $a - $b
 
        if ($a * $a + $b * $b -eq $c * $c) {
            $found = $true;
            break;
        }
    }
 
    if ($found) {
        break;
    }
}
"The Pythagorean triplet is ({0}, {1}, {2})" -f $a, $b, $c
"The product is {0}" -f ($a*$b*$c)

# The Answer
31875000
```