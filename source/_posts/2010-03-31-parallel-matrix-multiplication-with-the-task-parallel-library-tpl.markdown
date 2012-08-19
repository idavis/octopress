---
layout: post
title: "Parallel Matrix Multiplication with the Task Parallel Library (TPL)"
date: 2010-03-31 17:45
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
About

I brushed off some old benchmarking code used in my clustering application and decided to see what I can do using today’s multi-core hardware. When writing computationally intensive algorithms, we have a number of considerations to evaluate. The best (IMHO) algorithms to parallelize are [data parallel](http://en.wikipedia.org/wiki/Data_parallelism) algorithms without loop carried dependencies.

You may think nothing is special about matrix multiplication, but it actually points out a couple of performance implications when writing CLR applications. I originally wrote seven different implementations of matrix multiplication in C# – that’s right, seven.

+ Dumb
+ Standard
+ Single
+ Unsafe Single
+ Jagged
+ Jagged from C++
+ Stack Allocated

#### Dumb: double[N,N], real type: float64[0...,0...]

The easiest way to do matrix multiplication is with a .NET multidimensional array with i,j,k ordering in the loops. The problems are twofold. First, the i,j.k ordering accesses memory in a hectic fashion causing data in varied locations to be pulled in. Second, it is using a multidimensional array. Yes, the .NET multidimensional array is convenient, but it is very slow. Let’s look at the C# and IL

C#:
``` csharp
C[i, j] += A[i, k] * B[k, j];
```
IL of C#:
```
ldloc.s i
ldloc.s jcall instance float64& float64[0...,0...]::Address(int32, int32)
dup
ldobj float64
ldloc.1
ldloc.s i
ldloc.s k
call instance float64 float64[0...,0...]::Get(int32, int32)
ldloc.2
ldloc.s k
ldloc.s j
call instance float64 float64[0...,0...]::Get(int32, int32)
mul
add
stobj float64
```
If you notice the ::Address and ::Get parts, these are method calls! Yes, when you use a multidimensional array, you are using a class instance. So every access, assignment, and read incurs the cost of a method call. When you are dealing with and N^3 algorithm, that is N^3 method calls making this implementation much slower than other methods.

#### Standard: double[N,N], real type float64[0...,0...]

This implementation rearranges the loop ordering to i,k,j in order to optimize memory access to the arrays. No other changes are made from the dumb implementation. The Standard implementation is what is used for the base of all other multidimensional implementations.

#### Single: double[N * N], real type float64[]

Instead of creating a multidimensional array, we create a single block of memory. The float64[] type is a block of memory instead of a class. Downside here is that we have to calculate all offsets manually.

#### Unsafe Single, real type float64[]

This method is the same as the single dimensional array, except that the pointers to the arrays are fixed and pointers are used in unsafe C#.

#### Jagged, double[N][N], real type float64[][]

This is the same implementation as standard, except that we use arrays of arrays instead of a multidimensional array. It takes an extra step to initialize, but it is a series of blocks to raw memory eliminating the method call overhead. It is typically 30% faster that the multidimensional array.

#### Jagged from C++, double[N][N], real type float64[][]

This is a bit more difficult. When writing these algorithms, we let the JIT compiler optimize for us. The C++ compiler is unfortunately a lot better, but it isn’t real-time. I ported the code from the jagged implementation to C++/CLI and enabled heavy optimization. Once compiled, I disassembled the dll and converted the IL to C#. The result is this implementation which is harder to read, but it is really fast.

#### Stack Allocated, stackalloc double[N * N], real type float64*

This implementation utilizes the rarely used stackalloc keyword. Using this implementation is very problematic as you may get a StackOverflowException depending on your current stack usage.

Enough, what does it mean?!

Because of the way matrix multiplication works, we can multiply a row without impacting the result of other row multiplications. That’s right, we have a data parallel algorithm! We need to identify the chucks of work to parallelize.

We can go about this in a couple different ways. First, manual threading. Second, using the ThreadPool. Third, using the new task parallel library. In this post I am going to use the third option as it makes things very nice for us. We don’t want to deal with scheduling or synchronization ourselves.

Partitioning

We can do a couple of different schemes. We can use ParallelFor and add each rows multiplication to the list of work to be done. This is very easy
``` csharp
for ( int i = 0; i < N; i++ )
{
    for ( int k = 0; k < N; k++ )
    {
        for ( int j = 0; j < N; j++ )
        {
            double[] Ci = C[i];
            Ci[j] = ( A[i][k] * B[k][j] ) + Ci[j];
        }
    }
}
```
Becomes:
``` csharp
Parallel.For( 0, N, i =>
                    {
                        for ( int k = 0; k < N; k++ )
                        {
                            for ( int j = 0; j < N; j++ )
                            {
                                double[] Ci = C[i];
                                Ci[j] = ( A[i][k] * B[k][j] ) + Ci[j];
                            }
                        }
                    } );
```
The bad news is that this can be too granular. Another way to look at it is over parallelization. The overhead of scheduling and threading hurts our performance. This method does give us the smallest synchronization time. In the worst case, all rows are calculated except for one, which is started at the completion of the penultimate calculation. So the worst case is we wait for a single row.

If we are willing to put in more work, we can optimize the row striping to group the calculations into groups of rows in order to minimize synchronization and threading overhead. Looking at the number of processors available, we can make the number of chunks relative to the number of processors.
``` csharp
IEnumerable<Tuple<int, double[][]>> PartitionData( int N, double[][] A )
{
    int pieces = ( ( N % ChunkFactor ) == 0 )
                     ? N / ChunkFactor
                     : ( (int) ( ( N ) / ( (float) ChunkFactor ) ) + 1 );
 
    int remaining = N;
    int currentRow = 0;
 
    while ( remaining > 0 )
    {
        if ( remaining < ChunkFactor )
        {
            ChunkFactor = remaining;
        }
 
        remaining = remaining - ChunkFactor;
        var ai = new double[ChunkFactor][];
        for ( int i = 0; i < ChunkFactor; i++ )
        {
            ai[i] = A[currentRow + i];
        }
 
        int oldRow = currentRow;
        currentRow += ChunkFactor;
        yield return new Tuple<int, double[][]>( oldRow, ai );
    }
}
```
Here we partner the row to start on in the result matrix C with the pointer to the head of the row of A to start on. We could use the jagged implementation, but for pure speed, I am sticking with the fastest implementation I have:
``` csharp
void Multiply( Tuple<int, double[][]> A, double[][] B, double[][] C )
{
    int size = A.Item2.GetLength( 0 );
    int cols = B[0].Length;
    double[][] ai = A.Item2;
 
    int i = 0;
    int offset = A.Item1;
    do
    {
        int k = 0;
        do
        {
            int j = 0;
            do
            {
                double[] ci = C[offset];
                ci[j] = ( ai[i][k] * B[k][j] ) + ci[j];
                j++;
            } while ( j < cols );
            k++;
        } while ( k < cols );
        i++;
        offset++;
    } while ( i < size );
}
```
It is a little more complicated to write this heavily optimized version, but it pays off. To actually do the multiplication is very simple now using the TPL:
``` csharp
Parallel.ForEach( PartitionData( N, A ), item => Multiply( item, B, C ) );
```
#### Enough! Show me the numbers!

OK. You have patiently waited and I will give you the results. The parallel numbers are for my quad core machine with 4GB RAM. The Y-Axis is millions of algorithmic steps per second. The X-Axis is the matrix size.

First, we can look at the performance of the various single-threaded implementations. As you can see, there is a 5x difference between the easiest way and my most optimized implementation that does not use unsafe code. So if you need to multiply matrices – use jagged arrays!

{% img https://lh4.googleusercontent.com/-PtHlEF8Ynl8/Tv03HxgLZFI/AAAAAAAABB0/Vv0FW1ZWK7I/s800/SingleThreaded_All.png %}

The bulge you see on the left when the matrix size is small is the result of the Cache Memory Swap Die (CMSD) envelope. When the data size is small, the system leverages cache which is the fastest. When the data becomes to large, it has to store the data in main memory which is still pretty fast. What you don’t see is the swap and die levels. When the data set becomes very large, the system must use the swap file and store data on disk which is levels of magnitude slower than memory. When the data set becomes too large, the hard drive thrashes and you are constantly paging data in. At this point, your performance dies.

There is a problem with the chunking. Depending on how many pieces, the load of the processors, and the worst case job completion, the performance can vary for the chunking implementation. Below you can see the relative performance of the optimized chunking implementation against the simple parallelization and my heuristic.

I ran the chunking algorithm with many different granule sizes and recorded the best and worst values to give a performance range. The overlap between actual and max is the CPU being used by other processes out of my control – one of the issues with granule calculation is that you can’t predict the future. One possible thing we could do is to utilize coroutines to adjust the granule size as time goes on in oder to adjust to machine load.

{% img https://lh5.googleusercontent.com/-C_6JmOLr1Y0/Tv03GsNR5hI/AAAAAAAABBs/NHI_--pYB34/s800/Parallel_All.png %}

At best, we are calculating over 4x faster than the best single-threaded implementation. Yes, better than 4x with 4 cores. We are taking advantage of caching when we row stripe the data. This performance becomes even more significant when running this code across a cluster – you can get over 5x with 4 machines. Unfortunately, I never had the chance to work with four quad core machines in a cluster to test out a massively parallel implementation.

{% img https://lh3.googleusercontent.com/-yVNE7mG43UQ/Tv03DLN5J5I/AAAAAAAABBk/J9Jf1ptEIn4/s800/All_Compared.png %}

#### Summary

As you can see, small changes in algorithm implementation can have quite adverse effects on performance. Also, parallelizing data parallel algorithms can be incredibly simple and provide a great performance boost with very little effort.

I would love to do some work showing the parallelization of wavefront, four square, and other data dependency schemes and their effect on performance and the effort needed to parallelize them, but we will see how much time I have. Another fun problem might be a massively parallel N-queens or Knights tour solver.

You can find all of the source code for these benchmarks on [my blog’s GitHub page](http://github.com/idavis/blog/tree/master/source/MatrixMultipplication/).
