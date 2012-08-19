---
layout: post
title: "Fibonacci LINQ Sequence Generator Revisited"
date: 2010-05-04 19:09
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
t was pointed out that the sequence generator [from the previous article](/2010/04/fibonacci-linq-sequence-generator/) could be written as an infinite sequence to allow .Skip(), .Take(), .ElementAt(), etc. Here is the infinite sequence version.
``` csharp
public static class Fibonacci
{
    public static IEnumerable<BigInteger> Generate()
    {
        yield return 1;
        yield return 1;
 
        BigInteger n1 = 1, n2 = 1;
        while(true)
        {
            BigInteger n3 = n1 + n2;
            n1 = n2;
            n2 = n3;
            yield return n3;
        }
    }
}
```