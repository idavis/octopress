---
layout: post
title: "Project Euler: Problems 1 through 6 in PowerShell"
date: 2012-08-25 20:00:58
comments: true
categories: 
published: false
---
The first problem is rather easy. We just need to generate a sequence, filter it, and them sum the results. Calculating sums in PowerShell is syntactically awkward as you can see in the different examples.

``` ps1 Add all the natural numbers below one thousand that are multiples of 3 or 5.
# using Measure-Object
(1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0} | Measure-Object -Sum).Sum

# using begin, process, end blocks
1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % -begin {$sum=0 } -process {$sum+=$_} -end { $sum}

# cleaning up a bit
1..999 | ? {$_ % 5 -eq 0 -or $_ % 3 -eq 0}| % {$sum = 0} {$sum += $_} {$sum}

# The answer
233168 
```

The [Fibonacci sequence][] is much more fun. You can get close to this implementation with `yield return` in C#, but the multiple assignment trick has yet to come to C# :(.

``` ps1 Find the sum of the even-valued terms in the Fibonacci sequence whose values do not exceed four million
function Get-FibonacciSequence {
  param([int]$max)
  0
  1
  for($i = $j = 1; $i -lt $max) {
    $i
    $i,$j = ($i + $j),$i
   }
}

filter IsEven { if($_ % 2 -eq 0) { $_ } }

((Get-FibonacciSequence 4000000) | IsEven | Measure-Object -Sum).Sum

# The answer
4613732 
```

Generating prime factors is another simple algorithm. We divide off all of the 2's, then we take off the 3's, but from there we skip two instead of one as we already pulled out all of the possible even factors.

``` ps1 What is the largest prime factor of the number 600851475143
function Get-PrimeFactors {
  param([long]$number)
  while($number % 2 -eq 0) {
     2
     $number /= 2
  }

  for($i = 3; $i -le [Math]::Sqrt($number); $i += 2) {
     while($number % $i -eq 0) {
        $number /= $i
        $i
     }
  }
  $number
}

Get-PrimeFactors 600851475143 | select -last 1

# The answer
6857
```

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

This next one is just too simple to describe the solution.

``` ps1 Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.
$sum = (1..100 | Measure-Object -Sum).Sum
$sumOfSquares = (1..100 | % {$_*$_} | Measure-Object -Sum).Sum
$sum * $sum - $sumOfSquares

# The Answer
25164150
```

  [Fibonacci sequence]: http://en.wikipedia.org/wiki/Fibonacci_number