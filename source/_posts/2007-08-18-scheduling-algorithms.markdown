---
layout: post
title: "Scheduling Algorithms"
date: 2007-08-18 13:57
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
Below is the scheduling algorithms code that I referenced in my previous post. I no longer have access to a multi core/processor machine so I cannot benchmark them properly right now.
``` csharp
using System;
using System.Collections.Generic;
 
namespace Functionals
{
    public enum SchedulingAlgorithms : int
    {
        FixedChunking = 0,
        GuidedSelfScheduling,
        Factoring
    }
 
    public interface ISchedulingAlgorithm
    {
        /// <summary>
        /// Gets the partition sizes.
        /// </summary>
        /// <returns></returns>
        List<int> GetPartitionSizes();
    }
 
    public abstract class SchedulingAlgorithmBase : ISchedulingAlgorithm
    {
        protected static readonly int ProcessorCount = Environment.ProcessorCount;
 
        #region ISchedulingAlgorithm Members
 
        /// <summary>
        /// Gets the partition sizes.
        /// </summary>
        /// <returns></returns>
        public abstract List<int> GetPartitionSizes();
 
        #endregion
    }
 
    public sealed class FixedChunking : SchedulingAlgorithmBase
    {
        private readonly int size;
        private readonly int partitions;
 
        /// <summary>
        /// Initializes a new instance of the <see cref="FixedChunking"/> class.
        /// </summary>
        /// <param name="size">The size.</param>
        /// <param name="partitions">The partitions.</param>
        public FixedChunking( int size, int partitions )
        {
            this.size = size;
            this.partitions = partitions;
        }
 
        #region ISchedulingAlgorithm Members
 
        /// <summary>
        /// Gets the partition sizes.
        /// </summary>
        /// <returns></returns>
        public override List<int> GetPartitionSizes()
        {
            int numberOfPartitions = Math.Min( partitions, size );
            List<int> granualPartitionSizes = new List<int>( numberOfPartitions );
            int partitionSize = ( size + numberOfPartitions - 1 ) / numberOfPartitions;
            for (int i = 0; i < numberOfPartitions; i++)
            {
                granualPartitionSizes.Add( partitionSize );
            }
 
            return granualPartitionSizes;
        }
 
        #endregion
    }
 
    public sealed class GuidedSelfScheduling : SchedulingAlgorithmBase
    {
        private readonly int size;
 
        /// <summary>
        /// Initializes a new instance of the <see cref="GuidedSelfScheduling"/> class.
        /// </summary>
        /// <param name="size">The size.</param>
        public GuidedSelfScheduling( int size )
        {
            this.size = size;
        }
 
        #region ISchedulingAlgorithm Members
 
        /// <summary>
        /// Gets the partition sizes.
        /// </summary>
        /// <returns></returns>
        public override List<int> GetPartitionSizes()
        {
            double processors = Math.Max( ProcessorCount, 2 );
            int partitions = 1;
            if ( size > 1 )
            {
                partitions = (int) Math.Ceiling( Math.Log( size, processors ) );
            }
            int remaining = size;
            List<int> granualPartitionSizes = new List<int>( partitions );
            for (int i = 0; i < partitions; i++)
            {
                int partitionSize = (int) Math.Ceiling( ( remaining / processors ) );
                remaining -= partitionSize;
                granualPartitionSizes.Add( partitionSize );
            }
 
            return granualPartitionSizes;
        }
 
        #endregion
    }
 
    public sealed class Factoring : SchedulingAlgorithmBase
    {
        private readonly int threshold = 1;
        private readonly int size;
        private readonly double factor;
 
        /// <summary>
        /// Initializes a new instance of the <see cref="Factoring"/> class.
        /// </summary>
        /// <param name="size">The size.</param>
        /// <param name="factor">The factor.</param>
        public Factoring( int size, double factor )
        {
            this.size = size;
            this.factor = factor;
        }
 
        /// <summary>
        /// Initializes a new instance of the <see cref="Factoring"/> class.
        /// </summary>
        /// <param name="size">The size.</param>
        /// <param name="factor">The factor.</param>
        /// <param name="threshold">The threshold.</param>
        public Factoring( int size, double factor, int threshold )
        {
            this.size = size;
            this.factor = factor;
            this.threshold = threshold;
        }
 
        #region ISchedulingAlgorithm Members
 
        /// <summary>
        /// Gets the partition sizes.
        /// </summary>
        /// <returns></returns>
        public override List<int> GetPartitionSizes()
        {
            int processors = Math.Max( ProcessorCount, 2 );
            int remaining = size;
            int granualSize = 1;
            List<int> granualPartitionSizes = new List<int>();
            while ( ( remaining > threshold ) &&
                    ( granualSize > 0 ) )
            {
                granualSize = (int) Math.Ceiling( ( remaining * factor ) / processors );
                remaining = remaining - ( granualSize * processors );
                if ( granualSize > 0 )
                {
                    for (int i = 0; i < processors; i++)
                    {
                        granualPartitionSizes.Add( granualSize );
                    }
                }
            }
            if ( remaining > 0 )
            {
                granualPartitionSizes.Add( remaining );
            }
            return granualPartitionSizes;
        }
 
        #endregion
    }
}
```