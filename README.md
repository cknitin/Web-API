# WebAPI

## IHostedService vs BackgroundService in Web API (C#)
In ASP.NET Core, IHostedService and BackgroundService are used for running background tasks. However, there are key differences between the two.

## 1. IHostedService
IHostedService is a general interface that allows you to manage the start and stop lifecycle of a background task. It requires you to implement StartAsync and StopAsync.

``
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
``

