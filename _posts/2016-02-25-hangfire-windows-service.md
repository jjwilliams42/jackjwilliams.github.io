---
layout: post
published: false
author: jackjwilliams
title: Hangfire Windows Service
date: "2016-02-25 23:30"
comments: false
category: "C#"
tags: 
  - Octopus
  - ASP.NET 4.5
  - MVC 5
subtitle: Pains of converting Hangfire to a Windows Service
modified: ""
mathjax: false
featured: false
---


I'm trying to keep these posts more focused and to the point, here we go ...

## Problem Statement

The problem with simply stuffing your Hangfire service into a Windows Service (maybe using Topshelf) is that beforehand
everything is run on the ASP.NET runtime. So some of the "Startup" stuff that comes with ASP.NET MVC doesn't necessarily happen
as you would expect. This costed me HOURS of debugging. Lets get to it.

### Problem #1: No IUserTokenProvider is registered.

NOTE: This is only when you need to do things like send out user registration tokens or password reset tokens in the background.
 
Out of the box, your UserManager.UserTokenProvider setup might look something like this:
{% highlight c# %}
var dataProtectionProvider = options.DataProtectionProvider;
if (dataProtectionProvider != null)
{
    manager.UserTokenProvider = 
        new DataProtectorTokenProvider<ApplicationUser>(dataProtectionProvider.Create("ASP.NET Identity"));
}
{% endhighlight %}

Our setup was a little different, in a partial Startup class we get and set the DataProtectionProvider off of the IAppBuilder,
then later check for it and set it similarly.

{% highlight c# %}
if (Startup.DataProtectionProvider != null)
{
    manager.UserTokenProvider = 
        new DataProtectorTokenProvider<ApplicationUser>(Startup.DataProtectionProvider.Create("ASP.NET Identity"));
}
{% endhighlight %}

But now that Hangfire is in it's own project and it's own Windows Service - this never happened (hence the error **No IUserTokenProvider is registered**). We have a lot of user creations / modifications
that we offload to the background.

**The fix?**

Use your own.

{% highlight c# %}
var provider = new DpapiDataProtectionProvider("MyApplicationName");
UserManager.UserTokenProvider = new DataProtectorTokenProvider<ApplicationUser>(provider.Create("UserToken"));
{% endhighlight %}

Reference: [Stackoverflow](http://stackoverflow.com/questions/22629936/no-iusertokenprovider-is-registered)

### Problem 2: SignalR

I like to provide the user with nice feedback and SignalR works great for giving users feedback and notifications for long running processes.

When running Hangfire in process with your ASP.NET application and injecting hubs into services using your favorite IoC, everything plays 
together nicely. When you move it to a Windows service - Hangfire doens't have the right context. This makes sense after thinking about it,
but at 2 A.M. it didn't make much sense.

**The fix?**

[Scaleout in SignalR](http://www.asp.net/signalr/overview/performance/scaleout-in-signalr)

I won't go into much detail as you will need to pick your scale out method, but I will tell you the basics of what I had to do to get going.

I chose Scaleout with SQL Server. In both your ASP.NET MVC Startup code and your Hangfire Server process code you have to tell which server to use.

Add this line in both projects:

#### ASP.NET MVC
{% highlight c# %}
GlobalHost.DependencyResolver.UseSqlServer(sqlConnectionString);
app.MapSignalrR();
{% endhighlight %}

#### Hangfire Service Project
{% highlight c# %}
GlobalHost.DependencyResolver.UseSqlServer(sqlConnectionString);
{% endhighlight %}

For now, I personally used the same database that my project is in simply because I don't need to scale that far out (yet). When ready I 
can update my cloud infrastructure, add a database and modify this connection string. Pretty sweet!



The project I've been working on lately is all grown up and it is high time to create some kind of job queue for recurring / delayed tasks. I decided to go with [Hangfire](http://hangfire.io/). This post will focus on the database initialization, IIS / Hangfire server setup and a clean way to manage RecurringJobs. Later (in another post) I'm going to detail the SimpleInjector setup as it was initially a nightmare getting Hangfire to play well with my favorite IoC container. 

### Database Setup with Local / Staging / Develop / Environments
One issue I had upon first deploying Hangfire was how to automate the creation of the Hangfire databases. With so many environments I did not want to re-use the same database and definitely didn't want to have to create each one manually.

The [documentation](http://docs.hangfire.io/en/latest/configuration/using-sql-server.html) states that:

> SQL Server objects are being installed **automatically** from the SqlServerStorage constructor by executing statements described in the Install.sql file (which is located under the tools folder in the NuGet package). Which contains the migration script, so new versions of Hangfire with schema changes can be installed seamlessly, without your intervention.

#### The Problem

But this **never** happened for me - locally I use a (LocalDb) file, and if the database didn't exist Hangfire barfed. Rightly so - I'm not sure if Hangfire is equipped to create a LocalDb (it most likely would work fine on full-blown SQL Server instances). But I want it to work on localdb / full-blown instances. Here is the error:

{% highlight xml %}
An attempt to attach an auto-named database for file C:\....\App_Data\Hangfire.mdf failed. A database with the same name exists, or specified file cannot be opened, or it is located on UNC share.
{% endhighlight %}

#### Solution
After mucking around for awhile I decided to use something I know quite a bit about: Entity Framework. EF makes it easy to create databases automatically.  All I need is a DbContext class:

{% highlight c# %}
public class HangfireContext : DbContext
{
    public HangfireContext() : base("name=HangfireContext")
    {
        Database.SetInitializer<HangfireContext>(null);
        Database.CreateIfNotExists();
    }
}
{% endhighlight %}

In all of my web.configs (Develop, Production, Staging, etc...) the connection string points to the correct server and database (Hangfire_Staging, Hangfire_Developer, (LocalDb) for local dev). This creates the database for me in any new environment I stand up, and works out of the box for all new developers! All I have to do is instantiate it.

##### DB Initialization
I used the HangfireBootstrapper.cs example from the [documentation](http://docs.hangfire.io/en/latest/deployment-to-production/making-aspnet-app-always-running.html), with some slight modifications:

{% highlight c# %}
public void Start()
{
    lock (_lockObject)
    {
        if (_started) return;
        _started = true;

        HostingEnvironment.RegisterObject(this);
        
        //This will create the DB if it doesn't exist
        var db = new HangfireContext();

        GlobalConfiguration.Configuration.UseSqlServerStorage("HangfireContext");
        
	// See the next section on why we set the ServerName
        var options = new BackgroundJobServerOptions()
        {
            ServerName = ConfigurationManager.AppSettings["HangfireServerName"]
        };

        _backgroundJobServer = new BackgroundJobServer(options);

        var jobStarter = DependencyResolver.Current.GetService<JobBootstrapper>();
		
        //See the Recurring Jobs + SimpleInjector section
        jobStarter.Bootstrap();
               
    }
}
{% endhighlight %}

### Multiple IIS Sites Hosted Under the Default Web Site
Our dev/staging site structure looks like this

- /APP_Develop/
- /APP_Staging/

When spinning up hangfire for the first time in each env - the only site that had any Hangfire servers listed on the servers tab was APP_Develop (because that one was done first). When I opened APP_Staging no servers were listed in the Hangfire Dashboard. I am guessing this is because all servers were on the same machine, with the same machine name (under the same Default Web Site, each under a virtual directory) so I had to name the servers accordingly. This fixes that:

{% highlight c# %}
var options = new BackgroundJobServerOptions()
{
	ServerName = ConfigurationManager.AppSettings["HangfireServerName"]
};
{% endhighlight %}

Each web.config names its server something like: Hangfire_Develop, Hangfire_Staging

#### Recurring Jobs + SimpleInjector

To create and register RecurringJobs easily we roll out some interfaces and wire them up using SimpleInjector. This makes adding new jobs as easy as adding a new implementation of IRecurringJob.

##### IRecurringJob Interface

{% highlight c# %}
public interface IRecurringJob
{
    void Work();

    string When { get; }
}
{% endhighlight %}

##### IRecurringJob Implementation

{% highlight c# %}
public class Daily_ExpireSomeEntityJob : IRecurringJob
{
    private readonly IRepository _repo;

    public Daily_ExpireSomeEntityJob(IRepository repo)
    {
        _repo = repo;
    }

    public void Work()
    {
        var now = DateTime.Now;
        
        _repo.GetAll<SomeEntity>()
        	.Where(x => x.ExpirationDate <= now)
            .Each(x => x.Expired = true);
    }

    public string When
    {
        get { return Cron.Daily(23, 50); }
    }
}
{% endhighlight %}

##### Container Registration

{% highlight c# %}
var recurringJobs = 
    appAssemblies
    .SelectMany(a => a.GetTypes())
    .Where(
        type =>
            typeof (IRecurringJob).IsAssignableFrom(type) && 
            !type.IsAbstract && 
            !type.IsGenericTypeDefinition && 
            !type.IsInterface);
            
container.RegisterAll<IRecurringJob>(recurringJobs);
{% endhighlight %}

##### JobBootstrapper

This is how to bootstrap all recurring jobs when the applicatoin starts.

{% highlight c# %}
public class JobBootstrapper
{
    private readonly IEnumerable<IRecurringJob> _recurringJobs;

    public JobBootstrapper(IEnumerable<IRecurringJob> recurringJobs)
    {
        _recurringJobs = recurringJobs;
    }

    public void Bootstrap()
    {
        var manager = new RecurringJobManager();

        foreach (var job in _recurringJobs)
        {
            var type = job.GetType();
            var method = type.GetMethod("Work");
            var j = new Job(type, method);

            manager.AddOrUpdate(type.Name, j, job.When);
        }
    }
}
{% endhighlight %}

Hope this helps some others get going with Hangfire! So far the team and I are loving it - good job guys! 

Questions / Comments? Send me an email. I'll eventually setup Disqus!