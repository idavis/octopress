---
layout: post
title: "Project Euler: Problem 14 in PowerShell"
date: 2012-08-26 22:28
comments: true
categories: 
---

``` ps1 Base Algorithm
$cache = @{}
function Solve {
  param([int]$value)
  if($value -eq 1) {return 1}
  if($cache.ContainsKey($value)) {
    return $cache[$value]
  } else {
    if($value % 2 -eq 0) { $nextValue = $value/2 } 
    else { $nextValue = $value*3 + 1 }
    $cache[$value] = 1 + (Solve $nextValue)
    return $cache[$value]
  }
}

function Solve-Problem14 {
  1..999999 | % {$max=0;$key=-1} { $value = Solve $_; if($value -gt $max) {$max=$value;$key=$_} } {$key}
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem14}).TotalSeconds
Write-Host "Solution: " (Solve-Problem14)

Elapsed Time (s): 884.657156
Solution: 837799
```



``` ps1 Removing Stop Condition with Memoization
$cache = @{}
function Solve {
  param([int]$value)
  if($cache.ContainsKey($value)) {
    return $cache[$value]
  } else {
    if($value % 2 -eq 0) { $nextValue = $value/2 } 
    else { $nextValue = $value*3 + 1 }
    $cache[$value] = 1 + (Solve $nextValue)
    return $cache[$value]
  }
}

function Solve-Problem14 {
  1..999999 | % {$cache[1]=1;$max=0;$key=-1} { $value = Solve $_; if($value -gt $max) {$max=$value;$key=$_} } {$key}
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem14}).TotalSeconds
Write-Host "Solution: " (Solve-Problem14)

Elapsed Time (s): 884.657156
Solution: 837799
```


``` ps1 Iterative solution
$cache = @{}

function Solve-It {
  param([int]$value)
  $stack = New-Object System.Collections.Generic.Stack[int] 600
  $stack.Push($value)
  while($stack.Count -gt 0) {
    $value = $stack.Peek()
    if($value % 2 -eq 0) { $nextValue = $value/2 } 
    else { $nextValue = $value*3 + 1 }
    if($cache.ContainsKey($nextValue)) {
      $cache[$value] = 1 + $cache[$nextValue]
      $stack.Pop() | out-null
    } else {
      $stack.Push($nextValue)
    }
  }
  return $cache[$value]
}

function Solve-Problem14-it {
  1..999999 | % {$cache[1]=1;$max=0;$key=-1} { $value = Solve-It $_; if($value -gt $max) {$max=$value;$key=$_} } {$key}
}

Write-Host "Elapsed Time (s): " (Measure-Command {Solve-Problem14-it}).TotalSeconds
Write-Host "Solution: " (Solve-Problem14-it)

Elapsed Time (s): 
Solution: 837799
```
