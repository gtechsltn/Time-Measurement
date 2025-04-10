# Time Measurement
```
using System;
using System.Diagnostics;

public class MyService
{
    public void MyMethod()
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Your method logic here
            Console.WriteLine("Doing some work...");

        }
        finally
        {
            stopwatch.Stop();
            Console.WriteLine($"Time elapsed in MyMethod: {stopwatch.ElapsedMilliseconds} ms");
        }
    }
}
```

## TimeLogger.cs
```
public static class TimeLogger
{
    public static void MeasureExecutionTime(Action action, string methodName)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            action();
        }
        finally
        {
            stopwatch.Stop();
            Console.WriteLine($"Time elapsed in {methodName}: {stopwatch.ElapsedMilliseconds} ms");
        }
    }

    public static T MeasureExecutionTime<T>(Func<T> func, string methodName)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            return func();
        }
        finally
        {
            stopwatch.Stop();
            Console.WriteLine($"Time elapsed in {methodName}: {stopwatch.ElapsedMilliseconds} ms");
        }
    }
}
```

## Usage of TimeLogger.cs
```
public class MyService
{
    public void MethodA()
    {
        TimeLogger.MeasureExecutionTime(() =>
        {
            // Your code here
            Console.WriteLine("Executing MethodA...");
        }, nameof(MethodA));
    }

    public void MethodB()
    {
        TimeLogger.MeasureExecutionTime(() =>
        {
            // Your code here
            Console.WriteLine("Executing MethodB...");
        }, nameof(MethodB));
    }

    public int GetNumber()
    {
        return TimeLogger.MeasureExecutionTime(() =>
        {
            // Simulate some work
            Thread.Sleep(200);
            return 123; // returning int
        }, nameof(GetNumber));
    }

    public string GetData()
    {
        return TimeLogger.MeasureExecutionTime(() =>
        {
            // Simulate some work
            Thread.Sleep(500);
            return "Hello World"; // returning string
        }, nameof(GetData));
    }
}

```

# Using an Interceptor (AOP-like Approach with Attributes):
```
```

## ASP.NET Core API Action Filter:
```
public class ExecutionTimeFilter : ActionFilterAttribute
{
    private Stopwatch _stopwatch;

    public override void OnActionExecuting(ActionExecutingContext context)
    {
        _stopwatch = Stopwatch.StartNew();
    }

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        _stopwatch.Stop();
        var elapsed = _stopwatch.ElapsedMilliseconds;
        Console.WriteLine($"Execution time for {context.ActionDescriptor.DisplayName}: {elapsed} ms");
    }
}
```

## Usage of ASP.NET Core API Action Filter:
```
[ExecutionTimeFilter]
public class MyController : ControllerBase
{
    public IActionResult Get()
    {
        // Your logic
        return Ok();
    }
}
```

# For ASP.NET Core Web API

Step 1 — Add Serilog Timing Middleware

```
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Process
dotnet add package Serilog.Enrichers.Thread
```

Step 2 — Create Middleware for Logging Time Elapsed

```
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();

            var elapsedMs = stopwatch.ElapsedMilliseconds;
            var requestPath = context.Request.Path;

            _logger.LogInformation("Request [{Method}] {Path} executed in {ElapsedMilliseconds} ms",
                context.Request.Method, requestPath, elapsedMs);
        }
    }
}
```

Step 3 — Register Middleware in Program.cs

```
var builder = WebApplication.CreateBuilder(args);

// Serilog Config
builder.Host.UseSerilog((context, services, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithEnvironmentName()
        .Enrich.WithThreadId()
        .Enrich.WithProcessId()
        .WriteTo.Console();
});

var app = builder.Build();

// Add Middleware here (before endpoints)
app.UseMiddleware<RequestTimingMiddleware>();

app.MapControllers();

app.Run();
```

Output example:

```
[INF] Request [GET] /api/files executed in 125 ms
[INF] Request [POST] /api/users executed in 310 ms
[INF] Request [GET] /api/roles executed in 78 ms
```

# Log Method Execution Time (Per Method / Service)

Decorator Pattern:

```
public class TimingDecorator<T> : DispatchProxy where T : class
{
    public T Target { get; set; } = default!;
    public ILogger<T> Logger { get; set; } = default!;

    protected override object? Invoke(MethodInfo targetMethod, object?[]? args)
    {
        var stopwatch = Stopwatch.StartNew();
        try
        {
            return targetMethod.Invoke(Target, args);
        }
        finally
        {
            stopwatch.Stop();
            Logger.LogInformation("Method {MethodName} executed in {ElapsedMilliseconds} ms",
                targetMethod.Name, stopwatch.ElapsedMilliseconds);
        }
    }
}
```

Layer | Method | Benefit
-- | -- | --
API Request Time | Middleware | Global Request Logging
Service Method Time | Decorator | Per Method Execution Time
