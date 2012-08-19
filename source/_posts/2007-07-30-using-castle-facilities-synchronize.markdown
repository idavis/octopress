---
layout: post
title: "Using Castle.Facilities.Synchronize"
date: 2007-07-30 12:06
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
I removed the last post as the formatting was horrible. Here again is my first demo app with the Synchronization facility. It is normally quite painful to try to make your WinForms application threadsafe. VS2005 made it much easier to see that you are doing something wrong. Your application will move along and then the debugger will break and you have a window with an InvalidOperationException in you face.

``` csharp
System.InvalidOperationException was unhandled
Message="Cross-thread operation not valid: Control 'richTextBox' accessed from a thread other than the thread it was created on."
Source="System.Windows.Forms
StackTrace:
at System.Windows.Forms.Control.get_Handle()
at System.Windows.Forms.RichTextBox.StreamIn(Stream data, Int32 flags)
at System.Windows.Forms.RichTextBox.StreamIn(String str, Int32 flags)
at System.Windows.Forms.RichTextBox.set_Text(String value)
at SafelyUpdatingWinForm.MainForm.UpdateTime() in C:SafelyUpdatingWinFormSafelyUpdatingWinFormMainForm.cs:line 38
at SafelyUpdatingWinForm.MainForm.UpdateLoop() in C:SafelyUpdatingWinFormSafelyUpdatingWinFormMainForm.cs:line 27
at System.Threading.ThreadHelper.ThreadStart_Context(Object state)
at System.Threading.ExecutionContext.Run(ExecutionContext executionContext, ContextCallback callback, Object state)
at System.Threading.ThreadHelper.ThreadStart()
```

The IDE is nice enough though to point you in the direction of a solution [How to: Make Thread-Safe Calls to Windows Forms Controls](http://msdn.microsoft.com/en-us/library/ms171728.aspx) (MSDN). They suggest that for any component you want to change that you create delegates and try to use Control.Invoke to make the call on the appropriate thread. This gets very bloated when you have a large and complex UI. To simplify things, Craig Neuwirt created a new facility to plug into the Castle [WindsorContainer](http://www.castleproject.org/container/index.html). It can automatically martial the calls to the UI thread for you and a whole lot more. [Roy Osherove](http://osherove.com/blog) has his own [implementation](http://osherove.com/blog/2007/5/19/easier-winform-ui-thread-safe-methods-with-dynamicproxy2-and.html) that uses the DynamicProxy2.

``` csharp Program.cs:
using System;
using System.Windows.Forms;
using Castle.Facilities.Synchronize;
using Castle.Windsor; 

namespace SafelyUpdatingWinForm
{
     internal static class Program
     {
         [STAThread]
         private static void Main()
         {
             Application.EnableVisualStyles();
             Application.SetCompatibleTextRenderingDefault( false ); 

             WindsorContainer container = new WindsorContainer();
             container.AddFacility( "sync.facility", new SynchronizeFacility() );
             container.AddComponent( "mainform.form.class", typeof (MainForm) ); 

             Application.Run( container.Resolve( "mainform.form.class" ) as Form );
         }
     }
}
```

And then in
``` csharp MainForm.cs
using System;
using System.ComponentModel;
using System.IO;
using System.Net;
using System.Text;
using System.Text.RegularExpressions;
using System.Threading;
using System.Windows.Forms;

namespace SafelyUpdatingWinForm
 {
    public partial class MainForm : Form
    {
        private Thread updateLoop;
        private bool loop = true;
        private const int delay = 2000;

        public MainForm()
        {
            InitializeComponent();
        }

        private void UpdateLoop()
        {
            while ( loop )
            {
                UpdateTime();
                Thread.Sleep( delay );
            }
        }

        protected virtual void UpdateTime()
        {
            string text = GetWebPage();
            Regex regex = new Regex( @"<b>dd:dd:dd<br>" );
            Match match = regex.Match( text );
            string time = match.Value.TrimStart( "<b>".ToCharArray() ).TrimEnd( "<br>".ToCharArray() );
            richTextBox.Text = time;
        }

        private static string GetWebPage()
        {
            string url = "http://www.time.gov/timezone.cgi?Eastern/d/-5";
            HttpWebRequest webRequest = WebRequest.Create( url ) as HttpWebRequest;
            webRequest.Method = "GET";
            using (WebResponse webResponse = webRequest.GetResponse())
            {
                using (StreamReader sr = new StreamReader( webResponse.GetResponseStream(), Encoding.UTF8 ))
                {
                return sr.ReadToEnd();
                }
            }
        }

        protected override void OnClosing( CancelEventArgs e )
        {
            loop = false;
            while ( updateLoop.IsAlive ) { }
            base.OnClosing( e );
        }

        protected override void OnLoad( EventArgs e )
        {
            updateLoop = new Thread( new ThreadStart( UpdateLoop ) );
            updateLoop.Name = "UpdateLoop";
            updateLoop.Start();
            base.OnLoad( e );
        }
    }
}
```