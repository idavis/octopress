---
layout: post
title: "Correctness of Parallel Code"
date: 2007-08-12 13:54
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
One of the big problems in writing parallel applications is code correctness. How do we know that our parallel implementations are correct? While multi-core correctness has been researched and documented a great deal, correctness for clustering code appears to still be in its infancy. Writing, debugging, and verifying the correctness of your applications remains incredibly difficult. Now, I am not talking about grid computing or web services. I am specifically referring to MPI and the like doing high performance computing.

When debugging a parallel application, a user must set breakpoints, attach to multiple processes, and then try to step through the applications in parallel. This becomes even more unwieldy if you are also trying to write the library/service and test your applications. I am not a fan of debugging, and parallel debugging just makes my skin crawl. So, what can one do?

When designing the implementation of SPPM I wanted to approach it in a way that would allow flexibility never seen in a parallel programming system. Starting at the highest level, I abstracted the service to a well defined interface. Given this interface, I can actually implement several versions of my service that can be swapped out at runtime. But why would I want multiple versions? Using a framework like Castle, I can specify implementations of my service in a configuration file for the WindsorContainer to resolve. Depending on how I want to test, I can change my service implementation. I have two implementations right now: the real service, and a local virtual service. The real service tells my library that when calls are made, try to talk to an external service which runs on every node in the cluster. The local virtual service is used when you want to test a master and worker application without needing a cluster or external service. From the applications’ point of view nothing changes. They make the same library calls, but the service implementation has changed allowing the programmer to focus on their application and not its dependencies.

Another set of work dealt with the library implementation. All calls were implemented with a composite pattern partnering the command pattern and template method pattern. All commands would request a pointer from a communication interface factory which would return a pointer to come kind of network transmission class. This class is switched out when the service implementation changes – it may get a TCP socket communicator, a remoting communicator, or, just a pointer to an object that routes calls into the local virtual service.

Up to this point, all of the features has been done with design patterns and object containers. To push things one step further, I had to solve a problem that has really bothered me. Why must I actually execute all of my parallel applications together in order to test them? Why must I run a service, master, and worker just to verify that the simplest case works? I have already shown how to test using just the master and worker by abstracting the service, so what about verifying the correctness of the master and worker independently of one another? Impossible? Why? External Dependencies? What about the service layer? Is it at all possible to apply TDD to HPC and clusters? Of coarse, but it is far from obvious.

Aspect-oriented programming will allow us to simulate our dependencies without changing our code. The TypeMock web site does a good job at illustrating this. Our problem breaks down to the need to simulate external dependencies. Write unit tests for your parallel applications, but where external library calls would be made, like a parallel library which talks to the cluster, mock the calls! Below is a small sample trying to mock the worker of a parallel matrix multiplication application. The worker application is supposed to get the B matrix, and a series of rows Ai from A, and compute a sub-result Ci which is sent back to the master. The Check method will be called when the Put call is made and intercepted, and the Ci matrix will be verified for correct contents. The term tuple tells the worker application to exit when it tries to get the next piece of A.
``` csharp
[Test]
public void WorkerTest() {
    // We use this call to tell TypeMock that the next creation of
    // the InjectionLibrary needs to be faked. TypeMock will replace
    // the instance that should have been created with its own
    // empty implementation. The mocks we set up below tell
    // TypeMock that we are going to have calls made by another component
    // and we need to insert fake results.
    Mock mock = MockManager.Mock(typeof(InjectionLibrary));

    // Fake the B call. When the Worker application calls
    // Slm.Get("B")
    mock.ExpectAndReturn("Get", 0).Args(new Assign("B"), new Assign(B));

    // fake the Ai call
    string tupleName = String.Format("A0:A{0}", N - 1);
    mock.ExpectAndReturn("Get", 0).Args(new Assign(tupleName, new Assign(A)));

    // Grab the put of Ci and validate the result
    // We tell TypeMock to call the Check method below to
    // validate the argument (Ci matrix).
    mock.ExpectCall("Put").Args(Check.CustomChecker(new ParameterCheckerEx(Check)));

    // Fake the term tuple
    mock.ExpectAndReturn("Get", 0).Args(new Assign("A0:A0"), new Assign(term));

    // This next line is mocked, the library is static
    // and the faked instance is shared.
    InjectionLibrary Slm = InjectionLibrary.Instance;

    // Call our worker's main method.
    // All of the calls that were mocked above will
    // have there calls hijacked.
    OCMMWorker.Main(null);
}
```

Here we have referenced a .NET application like an external DLL and invoked its Main method. Everything is run from the comfort of your TestFixture and you have verified one case in your worker application without the need of a cluster, master, or external/virtual service. Using these methods, we can write parallel cluster code while minimizing our need to debug. Why debug when you know it works?