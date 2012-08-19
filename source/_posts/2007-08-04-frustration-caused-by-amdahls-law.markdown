---
layout: post
title: "Frustration Caused by Amdahl’s ‘Law’"
date: 2007-08-04 13:16
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
I was looking over some OpenMP resources and I found this line on the Wikipedia page:
{% blockquote [] [http://en.wikipedia.org/wiki/OpenMP] [OpenMP] %}
A large portion of the program may not be parallelized by OpenMP, which sets theoretical upper limit of speedup according to Amdahl’s law.
{% endblockquote %}
I will leave my rant concerning Amdahl’s ‘law’ for another post, but I feel like I need to point something out. According to Amdahl’s law, linear speedup is the theoretical maximum speedup (using N processors would be N). While some have argued saying that caching effects allow for a superlinear speedup, it is always small S * N where S is a small multiplier.

Wouldn’t you like to put the ‘super’ back into superlinear? But oh wait, Amdahl says I say you can do it. Crack open that MIT bible on algorithms ( I know you have/ had it) and find the section on decision problems. If you no longer have the book, take a quick look at [Wikipedia](http://en.wikipedia.org/wiki/Decision_problem).

In running code in parallel you still have the worst case near linear speedup. But, what happens when you are partitioning? What if you partition the data into two pieces, one for each processor? If there is no solution, you will run both processors exhaustively and you have the worst case. Now, let there be a solution to your decision problem. We have a few scenarios:

The solution is:

1. In the end of the first chunk, no speedup over non parallel.
2. In the end of the second chunk, near linear speedup.
3. In the beginning of the first chunk, no speedup over non parallel.
4. Anywhere other than the end of the second chunk – linear to true superlinear speedup!!!

As your solution in the problem moves toward the beginning of the chunk you can achieve amazing results. In working on Synergy and SPPM I have seen a speedup of over 2000 using two machines doing the 0/1 knapsack!

I still need to look into OpenMP some more to see if I can get it to shortcut the thread synchronization, but a true superlinear capable OpenMP application would be great. I made a number of modifications to the code from [Joe Duffy](http://www.bluebytesoftware.com/blog/)'s MSDN article “[Reusable Parallel Data Structures and Algorithms: Reusable Parallel Data Structures and Algorithms](http://msdn.microsoft.com/msdnmag/issues/07/05/CLRInsideOut/default.aspx)” to implement scheduling algorithms for loop tiling.  I have fixed chunking, guided self-scheduling, and factoring algorithms implemented. I am also hoping to add a synchronization scheme to allow one thread to tell the rest to stop when a solution is found. Once complete I should have a load balanced multi-core loop tiling library capable of superlinear speedup.