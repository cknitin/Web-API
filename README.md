# WebAPI

## IHostedService vs BackgroundService in Web API (C#)
In ASP.NET Core, IHostedService and BackgroundService are used for running background tasks. However, there are key differences between the two.

## 1. IHostedService
IHostedService is a general interface that allows you to manage the start and stop lifecycle of a background task. It requires you to implement StartAsync and StopAsync.

```
public class MyHostedService : IHostedService
{
    private Timer _timer;

    public Task StartAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedService is starting...");
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));
        return Task.CompletedTask;
    }

    private void DoWork(object state)
    {
        Console.WriteLine($"Task running at: {DateTime.Now}");
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        Console.WriteLine("MyHostedService is stopping...");
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }
}
```
## Register it in Program.cs

```
builder.Services.AddHostedService<MyHostedService>();
```

## 2. BackgroundService
BackgroundService is an abstract class that implements IHostedService but provides an easier way to run long-running background tasks using an ExecuteAsync method.

### Example using BackgroundService
    
```
public class MyBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine("MyBackgroundService is starting...");
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine($"Task running at: {DateTime.Now}");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

```

### Register it in Program.cs

```
builder.Services.AddHostedService<MyBackgroundService>();
```

### Key Differences

### When to Use What?
- Use IHostedService when:
    - You need lifecycle events (StartAsync, StopAsync).
    - You want to control when the task starts and stops.
- Use BackgroundService when:
    - You need continuous or periodic background execution.
    - You want a cleaner implementation without handling manual tasks.
    - Would you like me to tailor an example for a specific use case, such as processing messages from a queue or scheduled tasks? 🚀

