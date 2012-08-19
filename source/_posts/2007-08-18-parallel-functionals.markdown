---
layout: post
title: "Parallel Functionals"
date: 2007-08-18 14:02
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
I have used the loop tiling code with scheduling algorithms to parallelize [Dustin Campbell](http://diditwith.net/)’s code from his post [A Higher Calling](http://diditwith.net/2007/06/21/AHigherCalling.aspx). The parallelized functionals can be found below. It uses the scheduling algorithm code from the last post. The parallel reduction is actually based on Joe Duffy’s serial reduction.
``` csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading;
 
namespace Functionals
{
    public static class Parallel
    {
        private static readonly int ProcessorCount = Environment.ProcessorCount;
 
        [ThreadStatic]
        private static int partitions = ProcessorCount;
 
        public static int Partitions
        {
            get { return partitions; }
            set { partitions = value; }
        }
 
        [ThreadStatic]
        private static SchedulingAlgorithms algorithm = SchedulingAlgorithms.Factoring;
 
        public static SchedulingAlgorithms Algorithm
        {
            get { return algorithm; }
            set { algorithm = value; }
        }
 
        [ThreadStatic]
        private static double factor = 0.5;
 
        public static double Factor
        {
            get { return factor; }
            set { factor = value; }
        }
 
        [ThreadStatic]
        private static int factoringThreshold = 1;
 
        public static int FactoringThreshold
        {
            get { return factoringThreshold; }
            set { factoringThreshold = value; }
        }
 
        /// <summary>
        /// Filters the specified source as defined in <see cref="Functionals.Serial"/> except
        /// that the operation is done in parallel with the specified number of simultaneous operations.
        /// </summary>
        /// <typeparam name="TSource">The type of the source.</typeparam>
        /// <param name="source">The source.</param>
        /// <param name="predicate">The predicate.</param>
        /// <returns></returns>
        public static IList<TSource> Filter<TSource>( IList<TSource> source, Predicate<TSource> predicate )
        {
            Debug.Assert( partitions >= 1 );
 
            List<TSource> result = new List<TSource>();
            if ( source != null )
            {
                For( source, delegate( TSource item )
                            {
                                if ( predicate( item ) )
                                {
                                    result.Add( item );
                                }
                            } );
            }
 
            return result;
        }
 
        /// <summary>
        /// Map transforms each item with a conversion function as defined in
        /// <see cref="Functionals.Serial.Map"/> except
        /// that the operation is done in parallel with the specified number of simultaneous operations.
        /// </summary>
        public static IList<TResult> Map<TSource, TResult>( IList<TSource> source,
                                                            Converter<TSource, TResult> converter )
        {
            Debug.Assert( partitions >= 1 );
 
            List<TResult> result = new List<TResult>();
 
            if ( source != null )
            {
                For( source, delegate( TSource item ) { result.Add( converter( item ) ); } );
            }
 
            return result;
        }
 
        /// <summary>
        /// Reduces the specified source using the given accumlator as defined in
        /// <see cref="Functionals.Serial.Reduce"/> except
        /// that the operation is done in parallel with the specified number of simultaneous operations.
        /// </summary>
        /// <typeparam name="TSource">The type of the source.</typeparam>
        /// <param name="source">The source.</param>
        /// <param name="startValue">The start value.</param>
        /// <param name="accumulator">The accumulator.</param>
        /// <returns></returns>
        public static TSource Reduce<TSource>( IList<TSource> source, TSource startValue, Accumulator<TSource, TSource> accumulator )
        {
            Debug.Assert( partitions >= 1 );
            List<int> gp = GetPartitionList( source.Count );
            partitions = gp.Count;
 
            TSource[] partialReductionResults = new TSource[partitions];
            int partitionSize = ( source.Count + partitions - 1 ) / partitions;
            For( 0, partitions, delegate( int index )
                                {
                                    partialReductionResults[index] = default( TSource );
                                    int end = Math.Min( source.Count, partitionSize * ( index + 1 ) );
                                    for (int j = partitionSize * index; j < end; j++)
                                    {
                                        partialReductionResults[index] = accumulator( partialReductionResults[index], source[j] );
                                    }
                                } );
 
            // Do the final reduction on the master thread.
            return Serial.Reduce( partialReductionResults, startValue, accumulator );
        }
 
        /// <summary>
        /// Implements a parallel for loop of the form
        /// <code>
        /// for(int index = from; index &lt; to; index++)
        /// {
        ///    indexBoundClause(index);
        /// }
        /// </code>
        /// which will attempt to run in partitions equal to the number of processors.
        /// </summary>
        /// <param name="from">The start index.</param>
        /// <param name="to">the end index + 1.</param>
        /// <param name="indexBoundClause">The index bound clause to execute.</param>
        public static void For( int from, int to, Action<int> indexBoundClause )
        {
            For<int>( null, from, to, null, indexBoundClause );
        }
 
        /// <summary>
        /// Implements a parallel for loop of the form
        /// <code>
        /// for(int index = 0; index &lt; data.Count; index++)
        /// {
        ///    dataBoundClause(data[index]);
        /// }
        /// </code>
        /// which will attempt to run in partitions equal to the number of processors.
        /// </summary>
        /// <typeparam name="TInput">The type of the input.</typeparam>
        /// <param name="data">The data.</param>
        /// <param name="dataBoundClause">The data bound clause which is fed pieces of data.</param>
        public static void For<TInput>( IList<TInput> data, Action<TInput> dataBoundClause )
        {
            For( data, 0, data.Count, dataBoundClause, null );
        }
 
        /// <summary>
        /// Implements a parallel for loop that is either data or index bound. Depending on the
        /// overload that was called we will either be iterating over data or acting on an index.
        /// Either way we determine the bounds of our loop and partition the work using delegates
        /// and the ThreadPool. Once all pooled threads have finished this call will exit.
        /// </summary>
        /// <typeparam name="TInput">The type of the input.</typeparam>
        /// <param name="data">The data to pass into the dataBoundClause.</param>
        /// <param name="from">The starting index.</param>
        /// <param name="to">The ending index + 1.</param>
        /// <param name="dataBoundClause">The data bound clause.</param>
        /// <param name="indexBoundClause">The index bound clause.</param>
        private static void For<TInput>( IList<TInput> data, int from, int to, Action<TInput> dataBoundClause, Action<int> indexBoundClause )
        {
            Debug.Assert( from < to );
            Debug.Assert( ( dataBoundClause != null ) || ( indexBoundClause != null ) );
            Debug.Assert( ( data != null && dataBoundClause != null ) || ( data == null && dataBoundClause == null ) );
 
            int size = to - from;
            int offset = 0;
            List<int> gp = GetPartitionList( size );
 
            int parts = gp.Count;
            CountdownLatch latch = new CountdownLatch( parts );
            for (int i = 0; i < parts; i++)
            {
                int start = offset + from;
                int partitionSize = gp[0];
                gp.RemoveAt( 0 );
                int end = Math.Min( to, start + partitionSize );
                offset += partitionSize;
 
                Debug.WriteLine( string.Format( "From: {0}\tTo: {1}\tTotal: {2}", start, end, end - start ) );
 
                ThreadPool.QueueUserWorkItem( delegate
                                              {
                                                  for (int j = start; j < end; j++)
                                                  {
                                                      if ( data != null )
                                                      {
                                                          dataBoundClause( data[j] );
                                                      }
                                                      else
                                                      {
                                                          indexBoundClause( j );
                                                      }
                                                  }
 
                                                  latch.Signal();
                                              } );
            }
            latch.Wait();
        }
 
        /// <summary>
        /// Gets the partition list.
        /// </summary>
        /// <param name="size">The size.</param>
        /// <returns></returns>
        private static List<int> GetPartitionList( int size )
        {
            ISchedulingAlgorithm schedulingAlgorith = null;
 
            switch ( Algorithm )
            {
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
    }
}
```