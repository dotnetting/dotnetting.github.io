---
title: "How to use task-coalescing in C# to stop doing wasteful work"
date: 2025-01-08
description: "This post shows how to apply a technique known as task-coalescing to mitigate issues from cache stampede and stop doing wasteful work in multithreaded applications."
summary: "The post discusses a common inefficiency in multithreaded applications where multiple tasks redundantly compute the same result, particularly in scenarios involving high-concurrency requests to remote servers. It introduces the task-coalescing technique, which ensures that only one task fetches the data while others reuse the result, reducing wasteful work. Using .NET's `Lazy<T>` and `ConcurrentDictionary<TKey, TValue>`, the example implementation demonstrates how to achieve this efficiently, mitigating issues like **cache stampedes** and improving performance with minimal code changes."
tags: ["dotnet", "csharp", "concurrency", "task-coalescing", "task", "cache-stampede", "lazy", "concurrentdictionary"]
draft: false
---

## Problem
In a multithreaded application, when multiple tasks run concurrently, it is in most cases wasteful and inefficient for every task to compute the same result. A more efficient approach is for only one of those tasks, generally the first to start running, to do the work and produce the result. The other tasks can _sit_ and wait for the first one to complete and then share and reuse the same result.

Imagine an application that receives a high number of requests that require some data from a remote server. The remote server can be a REST API, a database or something else. In some cases we control both, the application and the remote server, and we have some flexibility to scale up and down both, to accommodate the traffic. In some other cases, we cannot control how the remote server scales, and have to adhere to their rate limiting policies. No matter what the case is, as a good practice we should always avoid doing wasteful work.

A common _solution_ for this kind of problem is throwing a cache to store and reuse the results from the remote server for a while, before going and fetching them again. The use of cache, although correct in some of these cases, comes with its own challenges, out of the scope of this discussion... mostly. There is one particular challenge that task-coalescing helps to mitigate: the [cache stampede](https://en.wikipedia.org/wiki/Cache_stampede).

Task-coalescing technique can be used to protect downstream services (including cache) and databases from sudden spikes that involve fetching the same data concurrently.

Let's examine the following code.

```csharp
// The content of this record is not relevant, it is intentionally oversimplified
public record WeatherForecast(string Location, int Temperature) { }

public class WeatherForecastService
{
    private static Dictionary<string, WeatherForecast> _forecast = new()
    {
        ["London"] = new WeatherForecast("London", Temperature: Random.Shared.Next(1, 40)),
        ["Sydney"] = new WeatherForecast("Sydney", Temperature: Random.Shared.Next(1, 40)),
        ["Havana"] = new WeatherForecast("Havana", Temperature: Random.Shared.Next(1, 40))
    };

    public async Task<WeatherForecast> GetWeatherForecast(string location)
    {
        Console.WriteLine("Fetching Weather Forecast for {0}...", location);

        // Simulate latency to retrieve remote data
        // This will be normally a HTTP Request to the WeatherForecast API
        await Task.Delay(1000); 

        return _forecast[location];
    }
}

var service = new WeatherForecastService();

// Assume these are concurrent requests from users
var requests = new List<Task<WeatherForecast>>
{
    service.GetWeatherForecast("London"),
    service.GetWeatherForecast("Sydney"),
    service.GetWeatherForecast("Havana"),
    service.GetWeatherForecast("London"),
    service.GetWeatherForecast("London"),
    service.GetWeatherForecast("Havana"),
};

var forecast = await Task.WhenAll(requests);
foreach (var f in forecast)
{
    Console.WriteLine(f);
}


```
The previous is a fairly standard implementation. If we run that code, we get the following output, showing that every task is going and fetching the same weather forecast data from the downstream service. A wasteful effort.

```
Fetching Weather Forecast for London...
Fetching Weather Forecast for Sydney...
Fetching Weather Forecast for Havana...
Fetching Weather Forecast for London...
Fetching Weather Forecast for London...
Fetching Weather Forecast for Havana...

WeatherForecast { Location = London, Temperature = 31 }
WeatherForecast { Location = Sydney, Temperature = 12 }
WeatherForecast { Location = Havana, Temperature = 15 }
WeatherForecast { Location = London, Temperature = 31 }
WeatherForecast { Location = London, Temperature = 31 }
WeatherForecast { Location = Havana, Temperature = 15 }

```
A better outcome would it be for only one of these requests per location, to go a fetch the forecast from the downstream service. For the other requests, they should wait and reuse the retrieved forecast.

## Solution
I'm going to implement a technique known as task-coalescing to achieve the better outcome described before. Instead of doing this from scratch, I will leverage two existing types in .NET to help me deal with the concurrent nature of this problem: [`Lazy<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.lazy-1) and [`ConcurrentDictionary<TKey, TValue>`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2).

I will use `Lazy<T>` to wrap the request tasks in my previous example. `Lazy<T>` implementation will ensure that only the first thread to start executing the `Task` factory will run. Any other threads will be blocked and wait for that result. To complement my implementation, I will use the thread-safe `ConcurrentDictionary` to temporarily store the `Lazy<Task<T>>` instances, until the tasks complete. This allows for any concurrent threads asking for the same value to _join_ the leading task and wait for the result. Let's see the code.

```csharp
public class CoalescedWeatherForecastService
{
    private static Dictionary<string, WeatherForecast> _forecast = new()
    {
        ["London"] = new WeatherForecast("London", Temperature: Random.Shared.Next(1, 40)),
        ["Sydney"] = new WeatherForecast("Sydney", Temperature: Random.Shared.Next(1, 40)),
        ["Havana"] = new WeatherForecast("Havana", Temperature: Random.Shared.Next(1, 40))
    };

    private readonly ConcurrentDictionary<string, Lazy<Task<WeatherForecast>>> _cachedLazyTasks = new();

    public Task<WeatherForecast> GetWeatherForecast(string location)
    {
        return _cachedLazyTasks.GetOrAdd(location, new Lazy<Task<WeatherForecast>>(() => Task.Run(async () =>
        {
            try
            {
                Console.WriteLine("Fetching Weather Forecast for {0}...", location);

                // Simulate latency to retrieve remote data
                // This will be normally a HTTP Request to the WeatherForecast API
                await Task.Delay(1000);

                return _forecast[location];
            }
            finally
            {
                _cachedLazyTasks.TryRemove(location, out var _);
            }
        }))).Value;
    }
}

var coalescedService = new CoalescedWeatherForecastService();

var coalescedRequests = new List<Task<WeatherForecast>>
{
    coalescedService.GetWeatherForecast("London"),
    coalescedService.GetWeatherForecast("Sydney"),
    coalescedService.GetWeatherForecast("Havana"),
    coalescedService.GetWeatherForecast("London"),
    coalescedService.GetWeatherForecast("London"),
    coalescedService.GetWeatherForecast("Havana"),
};

var coalescedForecast = await Task.WhenAll(coalescedRequests);
foreach (var f in coalescedForecast)
{
    Console.WriteLine(f);
}
```
As we run this new implementation, using the `CoalescedWeatherForecastService`, we get the following output.

```
Fetching Weather Forecast for London...
Fetching Weather Forecast for Havana...
Fetching Weather Forecast for Sydney...

WeatherForecast { Location = London, Temperature = 4 }
WeatherForecast { Location = Sydney, Temperature = 32 }
WeatherForecast { Location = Havana, Temperature = 32 }
WeatherForecast { Location = London, Temperature = 4 }
WeatherForecast { Location = London, Temperature = 4 }
WeatherForecast { Location = Havana, Temperature = 32 }
```
As we can see now the actual request to fetch the weather forecast is happening only once per location, and still all six tasks returned a result. No wasteful work 😌

## Final thoughts
The task-coalescing technique can be good for mitigating issues that arise from cache stampede and in general scenarios when computing expensive results with high concurrency. I'm not suggesting we should always use this technique, as its value depends on the specific scenario and the problem this aims to solve may have other preferred mitigating strategies. On the positive side, this technique can be easily implemented without big code refactoring or deploying additional infrastructure. It is one of those techniques that can throw big performance improvements in exchange for little time investment, what some people call quick wins 😊

Keep dotnetting...
