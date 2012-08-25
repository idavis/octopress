---
layout: post
title: "Sieve of Eratosthenes and Sieve of Atkin in PowerShell"
date: 2012-08-26 09:57
comments: true
categories: 
published: false
---
Generating prime numbers is a very slow operation, but it can still be fun. Using PowerShell, we can leverage some interesting language features to implement various sieves.

``` ps1 Sieve Of Eratosthenes
function Apply-SieveOfEratosthenes {
  param([int] $limit)
  $isprime = @{}
  2..$limit | ? { $isprime[$_] -eq $null } | % {
      $_
      $isprime[$_] = $true
      for ($i=$_*$_ ; $i -le $limit; $i += $_ ) { $isprime[$i] = $false }
    }
}

(Measure-Command {Apply-SieveOfEratosthenes 1001001}).TotalSeconds
# 43.66s
```

The sieve of Eratosthenes is a very common technique to generate primes. A much newer, and generally faster, algorithm is the sieve of Atkin. It is slower in PowerShell due to some simplifications I have made as well as not optimizing the polynomial calculations.

``` ps1 Sieve of Atkin - following pseudocode for a straightforward version of the algorithm:
function Apply-SieveOfAtkin {
  param([int] $limit)
  $isprime = @{}
  $rootLimit = [Math]::Sqrt($limit)
  1..$rootLimit | % {
    $x = $_
    1..$rootLimit | % {
      $y = $_ 
      $n = 4*$x*$x+$y*$y
      if(($n -le $limit) -and ($n % 12 -eq 1 -or $n % 12 -eq 5)) {
        $isprime[$n] = !$isprime[$n]
      }
      $n = 3*$x*$x+$y*$y
      if (($n -le $limit) -and ($n % 12 -eq 7)) {
        $isprime[$n] = !$isprime[$n]
      }
      $n = 3*$x*$x-$y*$y
      if (($x -gt $y) -and ($n -le $limit) -and ($n % 12 -eq 11)) {
        $isprime[$n] = !$isprime[$n]
      }
    }
  }
  5..$rootLimit | ? { $isprime[$_] } | % {
      $square = $_*$_
      for ($i=$square ; $i -le $limit; $i += $square ) { 
        $isprime[$i] = $false
      }
    }
  2
  3
  5..$limit | ? {$isprime[$_]}
}

(Measure-Command {Apply-SieveOfAtkin 1001001}).TotalSeconds
# 85.67s
```

This was very slow, but we can cut off a significant amount of time by rewriting the bottom two loops. To minimize our iteration, we only need to process entries that are not `$null` and have a `$true` value. As an optimization to the algorithm, I did not generate a list of size `$limit` with all items initialized to `$false` as the indexer returns a falsy value if the entry has not been set. We can again optimize when setting the multiples of squares as we only want to mark items that were marked as `$true` to `$false` whereas before we set the entry to `$false` every time thereby adding new entries to the dictionary.

``` ps1
(Apply-SieveOfAtkin 101) -join ", "

2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97, 101
```

``` ps1 Language Optimized Sieve of Atkin
function Apply-SieveOfAtkin {
  param([int] $limit)
  $isprime = @{}
  $rootLimit = [Math]::Sqrt($limit)
  1..$rootLimit | % {
    $x = $_
    1..$rootLimit | % {
      $y = $_ 
      $n = 4*$x*$x+$y*$y
      if(($n -le $limit) -and ($n % 12 -eq 1 -or $n % 12 -eq 5)) {
        $isprime[$n] = !$isprime[$n]
      }
      $n = 3*$x*$x+$y*$y
      if (($n -le $limit) -and ($n % 12 -eq 7)) {
        $isprime[$n] = !$isprime[$n]
      }
      $n = 3*$x*$x-$y*$y
      if (($x -gt $y) -and ($n -le $limit) -and ($n % 12 -eq 11)) {
        $isprime[$n] = !$isprime[$n]
      }
    }
  }
  5..$rootLimit | ? { $isprime[$_] } | % {
    $square = $_*$_
    for ($i=$square ; $i -le $limit; $i += $square ) { 
      # check if it exists first, assigning without checking is actually much slower
      # as it creates the element, but we don't need elements that are false.
      if($isprime.ContainsKey($i)) {
        $isprime[$i] = $false
      }
    }
  }
  2
  3
  $isprime.Keys | ? { $isprime[$_] } | sort 
}

(Measure-Command {Apply-SieveOfAtkin 1001001}).TotalSeconds
# 52.81s
```

With these small optimizations, the implementation of the sieve of Atkin is now only 21% slower instead of 96% slower. Either way, we have two fun ways to generate prime numbers in PowerShell.