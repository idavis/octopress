---
layout: post
title: "Running an elevated child process"
date: 2009-06-30 15:42
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
If you ever need to run a child process in an elevated state, you can easily set the Verbs property on the ProcessStartInformation to “runas” and the child process will ask for permission when started. You need to watch out the the user denying access however as a Win32Exception will be thrown with its NativeErrorCode = 1223. Here is a simple example launching the admin command prompt.
``` csharp
internal class Program
{
    private const int UserCanceled = 1223;
 
    private static void Main( string[] args )
    {
        var psInfo = new ProcessStartInfo
                     {
                         FileName = Environment.GetEnvironmentVariable ("ComSpec"),
                         UseShellExecute = true,
                         Verb = ( Environment.OSVersion.Version.Major >= 6 ) ? "runas" : string.Empty
                     };
 
        try
        {
            Process process = Process.Start( psInfo );
            Console.WriteLine( "Process has Id: {0}", process.Id );
            Console.WriteLine( "Waiting for app to exit." );
            process.WaitForExit();
        }
        catch ( Win32Exception ex )
        {
            if ( ex.NativeErrorCode != UserCanceled )
            {
                throw;
            }
            // permission was denied.
            Console.WriteLine( "The operation was canceled by the user." );
        }
 
        Console.WriteLine( "App exited, press enter to exit." );
        Console.ReadLine();
    }
}
```