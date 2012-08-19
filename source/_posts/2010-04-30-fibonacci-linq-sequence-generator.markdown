---
layout: post
title: "Fibonacci LINQ Sequence Generator"
date: 2010-04-30 18:56
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
I wanted to see what could be done with the Fibonacci sequence, but I didn’t have something that played nicely with LINQ. Here is my first cut:
``` csharp
public static class Fibonacci
{
    public static IEnumerable<BigInteger> Generate( int n )
    {
        return Generate( (long) n );
    }
 
    public static IEnumerable<BigInteger> Generate( long n )
    {
        yield return 1;
        if ( n == 1 )
        {
            yield break;
        }
        yield return 1;
        if ( n == 2 )
        {
            yield break;
        }
 
        BigInteger n1 = 1, n2 = 1;
        for ( BigInteger i = 3; i <= n; i++ )
        {
            BigInteger n3 = n1 + n2;
            n1 = n2;
            n2 = n3;
            yield return n3;
        }
    }
}
```
Then you can do all sorts of computations.
``` csharp
// Sum all
Fibonacci.Generate( 144 ).Aggregate( BigInteger.Add ); // 1454489111232772683678306641952
// Average them (via extension method below)
Fibonacci.Generate( 144 ).Average(); // (10100618828005365858877129458, remainder 0, total 144)
```
But why BigInteger? When using int, the 46th Fibonacci number is the last capable of being represented; making the data type a uint gets us the 47th. When using long, the 92nd is the last with the and with ulong the 93rd. With the BigInteger implementation, I can generate the 250,000th in about four seconds! And I can go much higher. If .NET had shipped with a BigFloat class I would have tried using the closed form solution which we could use to calculate the series in parallel. Having to sort may negate the speedup, but any solutions that don’t require order might benefit well.

There is another problem with using BigInteger; I can’t return a BigFloat from an .Average() calculation and BigInteger doesn’t support any kind of fraction support other than modulo. I wonder how much work it would be to create a BogFloat based on BigInteger.
``` csharp
public static class BigIntegerExtensions
{
    public static Tuple<BigInteger, BigInteger, BigInteger> Average( this IEnumerable<BigInteger> source )
    {
        BigInteger total = BigInteger.Zero;
        BigInteger elements = BigInteger.Zero;
        foreach ( BigInteger item in source )
        {
            total += item;
            elements += BigInteger.One;
        }
        BigInteger remainder;
        BigInteger average = BigInteger.DivRem( total, elements, out remainder );
        return new Tuple<BigInteger, BigInteger, BigInteger>( average, remainder, elements );
    }
}
```