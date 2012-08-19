---
layout: post
title: "Ninject MessageBroker Update"
date: 2008-11-29 15:19
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
It has been a long time (10 months) since I worked with the Ninject MessageBroker and a couple things have changed since my last post. In the old version you had to connect the MessageBroker as a kernel component by hand. Since then the extension environment has been flushed out a lot more. Now you can register the MessageBrokerModule when constructing your kernel object.
``` csharp
using System;
using System.Diagnostics;
using System.Net;
using System.Text.RegularExpressions;
using Ninject;
using Ninject.Extensions.MessageBroker;
 
namespace NinjectMessageBroker
{
    internal class Program
    {
        private static void Main()
        {
            // Intialize our injection kernel adding message broker functionality.
            using (var kernel = new StandardKernel(new MessageBrokerModule()))
            {
                // Get the event publisher. It reads the current time and fires an event
                var pub = kernel.Get<TimeReader>();
                Debug.Assert(pub != null);
 
                // Get the subscriber, it waits to get the current time and writes it to stdout
                var sub = kernel.Get<TimeWriter>();
                Debug.Assert(sub != null);
 
                // Verify that they were wired together
                Debug.Assert(pub.HasListeners);
                Debug.Assert(sub.LastMessage == null);
 
                // Get the current time. It should automatically let the TimeWriter know
                // without either of them ever knowing of one another.
                pub.GetCurrentTime();
 
                // Wait to exit.
                Console.ReadLine();
            }
        }
    }
 
    internal class TimeWriter
    {
        public string LastMessage { get; set; }
 
        [Subscribe("message://Time/MessageReceived")]
        public void OnMessageReceived(object sender, EventArgs<string> args)
        {
            LastMessage = args.EventData;
            Console.WriteLine(LastMessage);
        }
    }
 
    internal class TimeReader
    {
        public bool HasListeners
        {
            get { return (MessageReceived != null); }
        }
 
        [Publish("message://Time/MessageReceived")]
        public event EventHandler<EventArgs<string>> MessageReceived;
 
        /// <summary>
        /// Gets the current time and updates all subscribers.
        /// </summary>
        public virtual void GetCurrentTime()
        {
            string text = GetWebPage();
            var regex = new Regex(@"dd:dd:dd");
            MatchCollection matches = regex.Matches(text);
            string time = ((matches.Count == 2) ? matches[1] : matches[0]).Value;
            SendMessage(time);
        }
 
        /// <summary>
        /// Gets the contents of a web page as a string.
        /// </summary>
        /// <returns></returns>
        private static string GetWebPage()
        {
            const string url = "http://www.time.gov/timezone.cgi?Eastern/d/-5";
            var webClient = new WebClient();
            return webClient.DownloadString(url);
        }
 
        /// <summary>
        /// Sends the message to all subscribers in a threadsafe manner.
        /// </summary>
        /// <param name="message">The message.</param>
        public void SendMessage(string message)
        {
            EventHandler<EventArgs<string>> messageReceived = MessageReceived;
 
            if (messageReceived != null)
            {
                messageReceived(this, new EventArgs<string>(message));
            }
        }
    }
 
    public class EventArgs<TData> : EventArgs
    {
        public new static readonly EventArgs<TData> Empty;
 
        static EventArgs()
        {
            Empty = new EventArgs<TData>();
        }
 
        private EventArgs()
        {
        }
 
        public EventArgs(TData eventData)
        {
            EventData = eventData;
        }
 
        public TData EventData { get; private set; }
    }
}
```