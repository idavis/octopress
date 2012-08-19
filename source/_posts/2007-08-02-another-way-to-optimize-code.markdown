---
layout: post
title: "Another Way to Optimize Code"
date: 2007-08-02 12:57
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
In a previous post I showed the performance of several matrix multiplication implementations. As impressive as the JIT compiler is, I had an inkling that the C++/CLI compiler could do better. I compiled the jagged array (type[N][N]) in C# with optimization enabled. Here is the original source

``` csharp
public static TimeSpan Multiply(int N) {
    double[][] C = new double[N][];
    double[][] A = new double[N][];
    double[][] B = new double[N][];
    int i, j, k;

    for (i = 0; i < N; i++) {
        C[i] = new double[N];
        B[i] = new double[N];
        A[i] = new double[N];

        for (j = 0; j < N; j++) {
            C[i][j] = 0;
            B[i][j] = i*j;
            A[i][j] = i*j;
        }
    }

    DateTime now = DateTime.Now;

    for (i = 0; i < N; i++) {
        for (k = 0; k < N; k++) {
            for (j = 0; j < N; j++) {
                C[i][j] += A[i][k]*B[k][j];
            }
        }
    }

    return DateTime.Now - now;
}
```

Using Reflector on the dll, the only thing that the compiler does is moves the declaration of k into line 23. I used the C++/CLI plugin for Reflector to get the code into C++.  With a little modification we get this code:
``` c++
static TimeSpan Multiply(int N)
{
    int i;
    int j;
    array<array<double>^>^ C = gcnew array<array<double>^>(N);
    array<array<double>^>^ A = gcnew array<array<double>^>(N);
    array<array<double>^>^ B = gcnew array<array<double>^>(N);
    for (i = 0 ; (i < N); i++) {
        C[i] = gcnew array<double>(N);
        B[i] = gcnew array<double>(N);
        A[i] = gcnew array<double>(N);
        for (j = 0 ; (j < N); j++) {
            C[i][j] = 0;
            B[i][j] = (i * j);
            A[i][j] = (i * j);
        }
    }
    DateTime now = DateTime::Now;
    for (i = 0 ; (i < N); i++) {
        for (int k = 0 ; (k < N); k++) {
            for (j = 0 ; (j < N); j++) {
                C[i][j] = (C[i][j] + (A[i][k] * B[k][j]));
            }
        }
    }
    return ((TimeSpan) (DateTime::Now - now));
}
```

Compiling this code with
```
/Ox /Ob2 /Oi /Ot /Oy /GL /D “WIN32″ /D “NDEBUG” /D “_UNICODE” /D “UNICODE” /FD /EHa /MD /Yu”stdafx.h” /Fp”ReleaseCppDemo.pch” /Fo”Release\” /Fd”Releasevc80.pdb” /W3 /nologo /c /Zi /clr /TP /errorReport:prompt /FU “c:WINDOWSMicrosoft.NETFrameworkv2.0.50727System.dll”
```
We get an optimized matrix multiply. Again, reflecting the exe to get the C# code we get our new code which is is quite a bit faster:
``` csharp
public static TimeSpan Multiply(int N) {
    double[][] C = new double[N][];
    double[][] A = new double[N][];
    double[][] B = new double[N][];
    int index = 0;

    if (0 < N) {
        do {
            C[index] = new double[N];
            B[index] = new double[N];
            A[index] = new double[N];
            int column = 0;
            int value = 0;
            do {
                C[index][column] = 0;
                double num7 = value;
                B[index][column] = num7;
                A[index][column] = num7;
                column++;
                value = index + value;
            } while (column < N);
            index++;
        } while (index < N);
    }

    DateTime now = DateTime.Now;
    int i = 0;
    if (0 < N) {
        do {
            int k = 0;
            do {
                int j = 0;
                do {
                    double[] Ci = C[i];
                    Ci[j] = (A[i][k] * B[k][j]) + Ci[j];
                    j++;
                } while (j < N);
                k++;
            } while (k < N);
            i++;
        } while (i < N);
    }
    return (TimeSpan) (DateTime.Now - now);
}
```
This optimized code averages 182 MOPS on my machine compared to the original C# which only achieved 118 MOPS. This is a lot of work, but the speedup is amazing. I will try to put up some IL later to figure out exactly why the last version is so much faster.