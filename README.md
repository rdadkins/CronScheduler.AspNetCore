# CronScheduler.AspNetCore 
[![Build status](https://ci.appveyor.com/api/projects/status/wrme1wr6kgjp3a0o?svg=true)](https://ci.appveyor.com/project/kdcllc/cronscheduler-aspnetcore)

The goal of this library was to design a simple Cron Scheduling engine that is based on build-in Asp.Net Core  IHostedService interface.
It is much lighter than Quartz schedular and operates inside of any .NET Core GenericHost thus makes it simpler to setup and configure.
In addition `IStartupJob` was added to support async initialization before the IWebHost is ready to start. Sample project includes support for
making sure that Database is created before the application starts.

## Uses Crontab format for Jobs/Tasks schedules
This library supports up to 5 seconds job intervals in the Crontab format thank to [HangfireIO/Cronos](https://github.com/HangfireIO/Cronos) library.

You can use [https://crontab-generator.org/](https://crontab-generator.org/) to generated needed job/task schedule.

Cron expression is a mask to define fixed times, dates and intervals. The mask consists of second (optional), minute, hour, day-of-month, month and day-of-week fields. All of the fields allow you to specify multiple values, and any given date/time will satisfy the specified Cron expression, if all the fields contain a matching value.

                                           Allowed values    Allowed special characters   Comment

    ┌───────────── second (optional)       0-59              * , - /                      
    │ ┌───────────── minute                0-59              * , - /                      
    │ │ ┌───────────── hour                0-23              * , - /                      
    │ │ │ ┌───────────── day of month      1-31              * , - / L W ?                
    │ │ │ │ ┌───────────── month           1-12 or JAN-DEC   * , - /                      
    │ │ │ │ │ ┌───────────── day of week   0-6  or SUN-SAT   * , - / # L ?                Both 0 and 7 means SUN
    │ │ │ │ │ │
    * * * * * *

## Example CronSchedulerApp
The sample website provides with use-case scenario for this library.

Includes the following sample service:
```csharp
    public class TorahQuoteJob : IScheduledJob
    {
        public string CronSchedule { get; }

        public bool RunImmediately { get; }

        public string CronTimeZone { get; }

        private readonly TorahService _service;
        private readonly TorahSettings _options;

        public TorahQuoteJob(
            IOptions<TorahSettings> options,
            TorahService service)
        {
            _options = options.Value;
            CronSchedule = _options.CronSchedule; //set to 10 seconds in appsettings.json
            RunImmediately = _options.RunImmediately;
            CronTimeZone = _options.CronTimeZone;
            _service = service;
        }

        public async Task ExecuteAsync(CancellationToken cancellationToken)
        {
            var index = new Random().Next(_options.Verses.Length);
            var exp = _options.Verses[index];

            var result = await _service.GetVerses(exp, cancellationToken);

            TorahVerses.Current = result;
        }
    }
```

Then register this service within the `Startup.cs`
```csharp
    services.AddScheduler(builder =>
    {
        builder.AddJob<TorahQuoteJob,TorahSettings>();
        builder.UnobservedTaskExceptionHandler = UnobservedHandler;
    });
```

- Sample uses Microsoft.Extensions.Http.Polly extension library to make http calls every 10 seconds.

## `IStartupJobs` to assist with async jobs initialization before the application starts

There are many case scenarios to use StartupJobs for the IWebHost interface or IGenericHost. Most common case scenario is to make sure that database is created and updated.
This library makes it possible by simply doing the following:

- In the Program.cs file add the following:

```csharp
        public static async Task Main(string[] args)
        {
            var host = CreateWebHostBuilder(args).Build();

            // process any async jobs required to get the site up and running
            await host.RunStartupJobsAync();

            host.Run();
        }
```

- Register the startup job in `Program.cs` or in `Startup.cs` file.

```csharp
   public static IWebHostBuilder CreateWebHostBuilder(string[] args)
        {
            return WebHost.CreateDefaultBuilder(args)
                    .ConfigureServices(services =>
                    {
                        services.AddStartupJob<SeedDatabaseJob>();
                    })
                    .ConfigureLogging((context, logger) =>
                    {
                        logger.AddConsole();
                        logger.AddDebug();
                        logger.AddConfiguration(context.Configuration.GetSection("Logging"));
                    })
                    .UseStartup<Startup>();
        }
```
## Background Queues
In some instances of the application the need for queueing of the tasks is required. In order to enable this add the following in `Startup.cs`.

```csharp
    services.AddQueuedService();
```
Then add sample async task to be executed by the Queued Hosted Service.

```csharp
    
    public class MyService
    {
        private readonly IBackgroundTaskQueue _taskQueue;

        public MyService(IBackgroundTaskQueue taskQueue)
        {
            _taskQueue = taskQueue;
        }
        
        public void RunTask()
        {
            _taskQueue.QueueBackgroundWorkItem(async (token)=>
            {
                // run some task
                await Task.Delay(TimeSpan.FromSeconds(10), token);
            }});
        }
    }
```


## Special Thanks to
- [Maarten Balliauw](https://blog.maartenballiauw.be/post/2017/08/01/building-a-scheduled-cache-updater-in-aspnet-core-2.html) for the Asp.Net Core idea for the background hosted implementation.
- [3 ways to use HTTPClientFactory in ASP.NET Core 2.1](http://www.talkingdotnet.com/3-ways-to-use-httpclientfactory-in-asp-net-core-2-1/)

## Docker build
Utilizes [King David Consulting LLC DotNet Docker Image](https://github.com/kdcllc/docker/tree/master/dotnet)

```bash
    docker-compose -f "docker-compose.yml" -f "docker-compose.override.yml" up -d --build
```
