---
layout: post
title: "Overhead of Parallel Applications and Scheduling Algorithms"
date: 2007-08-11 13:46
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
When we typically try to measure the speedup of a parallelized algorithm, many people only calculate the serial vs. parallel running time (Sp = Tseq/Tpar). The effective speedup can more closely be approximated with the following equation to further define Tpar:
```
Tpar = Tcomp + Tcomm + Tmem + Tsync
Tpar is the total parallel running time.
Tcomp is the time spent in parallel computation on all machines / processors.
Tmem is the time spent in main memory or disk.
Tsync is the time required to synchronize.
```
All of these are easily quantifiable with the exception of Tsync. In an ideal world Tsync is zero, but this is never the case. A single slow processor/machine/data link can make a huge difference with Tsync. So, how can we manage it?

The most straightforward way to try to minimize Tsync it it to apply a fixed chunk load. MPI uses this method. This is fine unless you have a heterogeneous cluster or other applications are running on the processors/machines being used.

An extension of the fixed chunking is to create a normalized value for all of your computing resources. Then, let the computing resources get jobs whose size is comparable to their ability. Another way, say we have three machines whose computing powers are A:1, B:2, C:3 (C is three times as powerful as A). If we have twelve pieces of work to distribute, we can group them into six groups of two (total / sum of power). In the time C will compute three of the chunks, B will have done two and A will have done one. The entire job should wrap up with a minimal Tsync. This kind of work load balancing works very well for heterogeneous clusters.

Other scheduling methods had been tried in clusters; factoring and guided self-scheduling being two, but they seem to add too much overhead for the Tcomm. There has to be a balance in creating chunky communication and minimizing Tsync. They may however work as valid scheduling algorithms for multi-core processing. The Tcomm is greatly reduced compared to cluster computing.

Working again from [Joe Duffy](http://www.bluebytesoftware.com/blog/)’s MSDN article "[Reusable Parallel Data Structures and Algorithms: Reusable Parallel Data Structures and Algorithms](http://msdn.microsoft.com/msdnmag/issues/07/05/CLRInsideOut/default.aspx)", I have reworked the loop tiling code to support scheduling algorithms.
``` csharp
[ThreadStatic]
private static int partitions = ProcessorCount;

[ThreadStatic]
private static SchedulingAlgorithms algorithm = SchedulingAlgorithms.Factoring;

[ThreadStatic]
private static double factor = 0.5;

private static void For<TInput>( IList<TInput> data, int from, int to, Action<TInput> dataBoundClause, Action<int> indexBoundClause )
{
    Debug.Assert( from < to );
    Debug.Assert( ( dataBoundClause != null ) || ( indexBoundClause != null ) );
    Debug.Assert( ( data != null && dataBoundClause != null ) ||
                  ( data == null && dataBoundClause == null ) );

    int size = to - from;
    int offset = 0;
    List<int> gp = GetPartitionList( size );

    int parts = gp.Count;
    CountdownLatch latch = new CountdownLatch( parts );
    for (int i = 0; i < parts; i++) {
        int start = offset + from;
        int partitionSize = gp[0];
        gp.RemoveAt( 0 );
        int end = Math.Min( to, start + partitionSize );
        offset += partitionSize;
        ThreadPool.QueueUserWorkItem( delegate
                                      {
                                          for (int j = start; j < end; j++) {
                                              if ( data != null ) {
                                                  dataBoundClause( data[j] );
                                              }
                                              else {
                                                  indexBoundClause( j );
                                              }
                                          }

                                          latch.Signal();
                                      } );
    }
    latch.Wait();
}

private static List<int> GetPartitionList( int size ) {
    ISchedulingAlgorithm schedulingAlgorith = null;

    switch ( Algorithm ) {
        case SchedulingAlgorithms.FixedChunking:
            schedulingAlgorith = new FixedChunking( size, Math.Min( size, partitions ) );
            break;
        case SchedulingAlgorithms.GuidedSelfScheduling:
            schedulingAlgorith = new GuidedSelfScheduling( size );
            break;
        case SchedulingAlgorithms.Factoring:
            schedulingAlgorith = new Factoring( size, factor, factoringThreshold );
            break;
    }

    Debug.Assert( schedulingAlgorith != null );
    return schedulingAlgorith.GetPartitionSizes();
}
```
With this setup, all you have to do is set the algorithm that you would like to use along with the scheduling algorithm. The variables are thread static so that many threads can have their own algorithms, but since the for loop is blocking you only need one copy per thread.

It will take a little bit of tweaking for your application’s algorithm, but these should produce reasonable results. My next goal is to get them benchmarked under different scenarios.