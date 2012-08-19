---
layout: post
title: "Why are int[,], double[,], single[,], et alia so slow?"
date: 2007-07-30 12:41
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
I wrote a few matrix multiplication (MM) applications for benchmarking my parallel processing research. I was blown away by how slow the C# implementations were working. I tried to write the MM application a few ways. It turns out that the fastest is a jagged array, type[N][N], rather than type[N*N] or type[N,N]. I even tried to use unsafe C# to use pointers, but the type[N][N] was still by far the fastest. I am going to compare the type[N][N] and the type[N,N] here.

Here is the C# code for the type[N,N]. I am using int as it benchmarks faster and I spend less time waiting. I have removed the initialization of the arrays as it just added more code to analyze and isn’t needed for this analysis:

``` csharp
private static int[,] Standard() {
    Console.WriteLine( "Init" );

    int rows = 1500;
    int cols = 1500;

    int[,] first = new int[rows,cols];
    int[,] second = new int[rows,cols];
    int[,] third = new int[rows,cols];

    Console.WriteLine( "Calculate" );

    for (int i = 0; i < rows; i++ )
        for (int j = 0; j < cols; j++) 
            for (int k = 0; k < rows; k++)
                third[i, j] += first[i, k] * second[k, j];

    return third;
}
```
I have added the console output to identify the IL more easily.

Here is the type[N][N] version:
``` csharp
private static int[][] Jagged() {
    Console.WriteLine( "Init" );

    int rows = 1500;
    int cols = 1500;

    int[][] first = new int[rows][];
    int[][] second = new int[rows][];
    int[][] third = new int[rows][];

    for (int i = 0; i < rows; i++) {
        first[i] = new int[cols];
        second[i] = new int[cols];
        third[i] = new int[cols];
    } 

    Console.WriteLine( "Calculate" );

    for (int i = 0; i < rows; i++)
        for (int j = 0; j < cols; j++)
            for (int k = 0; k < rows; k++)
                third[i][j] += first[i][k] * second[k][j];

    return third;
}
```

On first glance it looks like we have a bit more initialization overhead for the type[N][N]. I compiled everything and reflected out the IL. I then stripped all locations to keep this as clean and simple as possible (and the diff cleaner). Though there is a difference. The type[N][N] uses ldelem and stelem to initialize while type[N,N] uses call instance void int32[0...,0...]::Set(int32, int32, int32). I think this is where we start seeing why the C# array is slower than the jagged version.

If we look at the IL for the initialization that takes care of everything up to the writing of “Calculate” to the console:

<table style="width:auto;"><tr><td><a href="https://picasaweb.google.com/lh/photo/W4r-BR6mFj4bfyfn7eFoINMTjNZETYmyPJy0liipFm0?feat=embedwebsite"><img src="https://lh5.googleusercontent.com/-_ekmWRZuKr4/Rq5rx6H3kbI/AAAAAAAABBc/CP5vsDK5BBs/s144/Init.jpg" height="auto" width="auto" /></a></td></tr><tr><td style="font-family:arial,sans-serif; font-size:11px; text-align:right">From <a href="https://picasaweb.google.com/111296262999630241623/Blog?authuser=0&feat=embedwebsite">Blog</a></td></tr></table>

On the type[N,N] side we are creating a new object on the evaluation stack and then popping from the stack and storing in a local variable – this is done for each the three arrays.

On the type[N][N] we push a new primitive array of int32 onto the stack and for each position we use the same call to create a new primitive array and replace the value on the stack at the given position.

Here is the grand conclusion to our IL:


<table style="width:auto;"><tr><td><a href="https://picasaweb.google.com/lh/photo/fVxRLH6Y_uKZbmVmse0NBNMTjNZETYmyPJy0liipFm0?feat=embedwebsite"><img src="https://lh5.googleusercontent.com/-RMJhZXAMguI/Rq5rx6H3kaI/AAAAAAAABBc/J9tatQG9RRM/s144/Calculate.jpg" height="auto" width="auto" /></a></td></tr><tr><td style="font-family:arial,sans-serif; font-size:11px; text-align:right">From <a href="https://picasaweb.google.com/111296262999630241623/Blog?authuser=0&feat=embedwebsite">Blog</a></td></tr></table>

As you can see, the calculations happen ‘almost’ identically. In the type[N,N] array we are making external calls to the runtime in order to get an address and set values whereas the type[N][N] simply issue a ldelem. Now, I wish there were a way to see exactly how long the Address and Get methods take, but the evidence is clear that ldelem is a lot faster.

I ran all of the matrix permutations for all of the types of arrays I could use. The ikj MM was the fastest for all. I benchmarked all of these on two machines (S01, S02). The machines were as identical as possible and nothing else running except background services. Each point was run three times and the average of the three was plotted. The matrix size (bottom) was NxN (square) and the left hand side is the number of algorithmic steps per second. For MM the multiplication, addition, and assignment are 1 operation.

<table style="width:auto;"><tr><td><a href="https://picasaweb.google.com/lh/photo/OK9-bTAUY0KqFR_fP3ophNMTjNZETYmyPJy0liipFm0?feat=embedwebsite"><img src="https://lh5.googleusercontent.com/-L3ftSQxFJHY/Rq5oc6H3kZI/AAAAAAAABBc/nQKNqpX7-WQ/s144/SerialMMBenchmarks.jpg" height="98" width="144" /></a></td></tr><tr><td style="font-family:arial,sans-serif; font-size:11px; text-align:right">From <a href="https://picasaweb.google.com/111296262999630241623/Blog?authuser=0&feat=embedwebsite">Blog</a></td></tr></table>

For a 3000×3000 matrix we are making millions or method calls using the type[N,N] matrix that the type[N,N] does. Why is the C# matrix implemented in the way? Please let me know if I missed anything or you have some insight.

