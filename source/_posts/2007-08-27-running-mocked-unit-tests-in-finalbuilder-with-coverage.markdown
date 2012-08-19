---
layout: post
title: "Running Mocked Unit Tests in FinalBuilder With Coverage"
date: 2007-08-27 14:07
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
I just read an article by Craig Murphy on running unit tests in FinalBuilder. I liked the article and it gives some good pointers. I wish he had written it a while ago, it would have been helpful to me. We use TypeMock which will not allow us to make our unit tests as simply. When I wrote our build scripts I took the same general approach he did.

Starting in the Main ActionList we defined a few variables that give the paths to the applications we use. I also created an ActionList call RunTest which I will go into detail below. For each DLL being tested a call to RunTest is made passing in the name of the DLL to be tested.

{% img https://lh3.googleusercontent.com/-CF7f1y67AkI/RtL_7VfnfQI/AAAAAAAABBc/pb44A4VzKb4/s800/MainActionList.png %}

In each call we set the CURRENTDLL ActionList Parameter with the name of the DLL (Animation,Audio,Core,..).

{% img https://lh3.googleusercontent.com/-ou18A-6WXGk/RtL_7VfnfPI/AAAAAAAABBc/lzlobPw-Zjc/s800/ActionListParameters.png %}

NCOVEROPTIONS holds the command-line options for ncover.console.exe. The same follows for NUNITOPTIONS and TMOCKRUNNEROPTIONS.  The rest of the ActionList Parameters are used as temporary variables allowing me to put together very complex command argument strings and parse XML into variables.

The RunTest ActionList is separated into two try…catch blocks. The first to run the tests and the second to process the results.

{% img https://lh4.googleusercontent.com/-ph5Pp8VMH48/RtL_7lfnfRI/AAAAAAAABBc/LDsA1ZU7f7I/s800/RunTestActionList.png %}

Here are what the commands are doing:

Set Variable NCOVEROPTIONS
```
//x “%BUILDDIR%%CURRENTDLL%Coverage.Xml” //l “%BUILDDIR%%CURRENTDLL%Coverage.Log”
```
Set Variable TESTDLLS
```
“%ROOTDIR%bin%CURRENTDLL%Tests%CURRENTDLL%.Tests.dll”
```
Set Variable NUNITOPTIONS
```
%TESTDLLS% /xml:”%BUILDDIR%%CURRENTDLL%TestResult.xml” /noshadow
```
Set Variable PARAMETERS
```
-first -link NCover %NCOVER% %NCOVEROPTIONS% %NUNIT% %NUNITOPTIONS%
```

Execute Program
```
%TMOCKRUNNER% %PARAMETERS%
```
-Wait For Completion, Log Output, and Hide Window are checked. I do not check the exit code. The second try block handles it.

The first sets up all of our variables and launches the TMockRunner application with all of our parameters. The second tries to read the test results into the TESTFAILURES variable. If the TMockRunner failed then the file will not exist and the Catch block will append our error message to indicate this. If there are test failures we read the failed test cases into a temporary error variable using XPath. This temporary variable is appended to the error message variable which is sent via email to our team. This allows us to find the exact tests that failed directly from our email.

That is basically it. Our CI server runs the FinalBuilder script with a few extra options and processes the NCover output to give us a nice web page.
