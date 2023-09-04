# API Rate Limiting using .NET

## Introduction

API requests consume resources such as: 

- network
- CPU
- memory
- storage.

Resources are consumed by apps that rely on them, and when an app makes too many requests for a single resource, it can lead to resource contention. Resource contention occurs when a resource is consumed by too many apps, and the resource is unable to serve all of the apps that are requesting it.

The amount of resources required to satisfy a request greatly depends on the user input and endpoint business logic. Also, consider the fact that requests from multiple API clients compete for resources. An API is vulnerable if at least one of the following limits is missing or set inappropriately (e.g., too low/high):

- Execution timeouts
- Max allocable memory
- Number of file descriptors
- Number of processes
- Request payload size (e.g., uploads)
- Number of requests per client/resource
- Number of records per page to return in a single request response

## Solutions

Rate limiting is the concept of limiting how much a resource can be accessed. For example, you may know that a database your app accesses can safely handle 1,000 requests per minute, but it may not handle much more than that.

One of the solution used in the past was to implement a [DelegationHandler](https://learn.microsoft.com/en-us/dotnet/core/extensions/http-ratelimiter) subclass.

NET 7 introduce rate limiting middleware in ASP.NET Core ([available here](https://www.nuget.org/packages/Microsoft.AspNetCore.RateLimiting)).

This [blog post](https://devblogs.microsoft.com/dotnet/announcing-rate-limiting-for-dotnet/#ratelimiter-apis) is a very well documented approach on how to use this solutions.

All happens here:

```csharp
var app = WebApplication.Create(args);

app.UseRateLimiter(new RateLimiterOptions()
    .AddConcurrencyLimiter(
        policyName: "get", 
        new ConcurrencyLimiterOptions(
            permitLimit: 2, 
            queueProcessingOrder: QueueProcessingOrder.OldestFirst, 
            queueLimit: 2))
    .AddNoLimiter(policyName: "admin")
    .AddPolicy(policyName: "post", partitioner: httpContext =>
    {
        if (!StringValues.IsNullOrEmpty(httpContext.Request.Headers["token"]))
        {
            return RateLimitPartition.CreateTokenBucketLimiter("token", 
                key => new TokenBucketRateLimiterOptions(
                    tokenLimit: 5, 
                    queueProcessingOrder: QueueProcessingOrder.OldestFirst,
                    queueLimit: 1, 
                    replenishmentPeriod: TimeSpan.FromSeconds(5), 
                    tokensPerPeriod: 1, 
                    autoReplenishment: true));
        }
        else
        {
            return RateLimitPartition.Create("default", key => new MyCustomLimiter());
        }
    }));

app.MapGet("/get", context => context.Response.WriteAsync("get")).RequireRateLimiting("get");

app.MapGet("/admin", context => context.Response.WriteAsync("admin")).RequireRateLimiting("admin").RequireAuthorization("admin");

app.MapPost("/post", context => context.Response.WriteAsync("post")).RequireRateLimiting("post");

app.Run();
```

## Reference

[OWASP rate Limiting](https://owasp.org/API-Security/editions/2019/en/0xa4-lack-of-resources-and-rate-limiting/)  
[Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0)  
[The throttling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/rate-limiting-pattern)  
[The throttling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/throttling)  
[Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-7.0)  
[System.Threading.RateLimiting](https://www.nuget.org/packages/System.Threading.RateLimiting)