---
layout: post
title: "Ruby First Impressions"
date: 2008-12-28 15:25
comments: true
categories: 
author: "Ian Davis"

# Github repositories
github_user: idavis
github_repo_count: 0
github_show_profile_link: true
github_skip_forks: true

# Twitter
twitter_user: ianfdavis
twitter_tweet_count: 4
twitter_show_replies: false
twitter_follow_button: true
twitter_show_follower_count: false
twitter_tweet_button: true
---
I decided to try out Ruby today. I have wanted to try it out for quite a while. Some of my impressions may be inaccurate, but I have done what I can. I have Zero interest in DB and web ‘programming’ so I am going to skip rails and related material.

What I like

+ very straightforward language
+ I loved that when I wanted to shift the elements of an array I found it was built-in
+ Class library
+ Being able to have a line of code like this; I wish I knew what to call it
+ + return SolveUsingClosedFormExpression( n ) if ( 0..1474 ).include? n
+ + return SolveUsingClosedFormExpression( n ) if (0..1474) === n
+ I will complain about this as well, but in writing the closed form Fibonacci solution, the data type changes allowing for huge results
``` ruby
def SolveUsingClosedFormExpression(n)
left = GOLDENRATIO ** n
right = (-GOLDENRATIOINV) ** n
  return ( (left – right) / ROOT5 ).to_i
end
```

What I didn’t like

+ Trying to figure out how to use gems (packages, not the tool) and files in the class library
+ Lack of type information
+ + Yes, yes, I know, I know, but for a c, c++/cli, c#  programmer, it feels just wrong
+ + Declaring a variable feels like I am introducing a magic variable – *poof* it exists. At least in TI-Basic I had to declare my dynamically typed variables.
+ + Lack of method return types
+ <del>At least four ways to return a value from a method, see below.</del>
+ Inconsistent API
+ +  was very frustrated when trying to use .power! only to find that it isn’t defined for Float types – I have to use ** everywhere.
+ <del>Ruby is supposed to be super OO, but what object contains puts/print?</del>

<del>What I really didn’t like</del>

``` ruby
def multiplyWithReturn(val1, val2 )
    result = val1 * val2
    return result
end
 
def multiplyLocalVariable(val1, val2 )
    result = val1 * val2
end
 
def multiplyNoVariables(val1, val2 )
    val1 * val2
end
 
def multiplyAssignToMethodName(val1, val2 )
    multiplyAssignToMethodName = val1 * val2
end
 
puts multiplyWithReturn(5,6)
puts multiplyLocalVariable(5,6)
puts multiplyNoVariables(5,6)
puts multiplyAssignToMethodName(5,6)
```

<del>Running this code prints 30303030</del>

<del>I don’t know if I am missing a key ‘feature’ of the language, but at least four ways to return a value from a method just really irks me. Given that there is no return type of a method, reading code to determine what is returning a value and what methods are void seems ridiculous.</del>

I have never programmed in a dynamic language, so it has been a bit of a ride. There a so many nuances to the language, it will take a while to get them down, but they allow for very concise code.