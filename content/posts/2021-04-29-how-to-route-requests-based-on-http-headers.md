---
title: "How to route requests based on HTTP headers in ASP.NET Core"
date: 2021-04-29
description: "This post shows how to route requests to specific actions in ASP.NET Core via implementing IActionConstraint and decorating actions and controllers with an Attribute to indicate the value of a header to determine the valid route to follow."
tags: ["asp.net core", "csharp", "routing", "http headers"]
draft: false
---

It is common for APIs to route requests based on a combination of HTTP verb (GET, POST, etc.) and path. For example, we can have same path `/api/items/123` for both retrieving the information about item `123` and deleting item `123`, using HTTP verbs `GET` and `DELETE` respectively. But what if we have the same verb and path and need to route the requests based on the value of a HTTP header?

## Problem

In some situations we may have to expose the same path and HTTP verb to support different behaviour. Imagine we are responsible for maintaining a microservice in a CRM system that uses the `Accept` header to handle versioning. We need to support the following requests:

> `Accept: application/vnd.crm.v1+json, GET api/customers/123`

> `Accept: application/vnd.crm.v2+json, GET api/customers/123`

Each of those requests produces different results so we want to handle them in different actions. Because the verb and the path are the same, we have to route the request to the right action based on the value of the `Accept` header.

## Solution

ASP.NET Core introduced the interface [`IActionConstraint`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.actionconstraints.iactionconstraint) that allows us to determine whether or not an action is applicable for a given request. We can create then an attribute and implement this interface to determine which method to execute based on the value of the `Accept` header.

```csharp
public class AcceptsAttribute : Attribute, IActionConstraint
{
    private readonly string _acceptHeaderValue;

    public AcceptsAttribute(string acceptHeaderValue)
    {
        _acceptHeaderValue = acceptHeaderValue;
    }

    public bool Accept(ActionConstraintContext context)
    {
        if (!context.RouteContext.HttpContext.Request.Headers.ContainsKey("Accept"))
        {
            return false;
        }

        var headerValue = context.RouteContext.HttpContext.Request.Headers["Accept"];
        return headerValue.Contains(_acceptHeaderValue);
    }

    public int Order => 0;
}
```

And then we can decorate our actions with the new `Accepts` attribute to indicate the supported version.

```csharp
[Route("api/customers")]
[ApiController]
public class CustomersController : ControllerBase
{
    [HttpGet]
    [Route("{id}")]
    [Accepts("application/vnd.crm.v1+json")]
    public IActionResult GetV1(int id)
    {
        // Implementation omitted for brevity
    }

    [HttpGet]
    [Route("{id}")]
    [Accepts("application/vnd.crm.v2+json")]
    public IActionResult GetV2(int id)
    {
        // Implementation omitted for brevity
    }
}
```

You can instead decorate a controller with this attribute, which would mean all the actions in the controller support that version.

## Final thoughts
In the previous example I used the `Accept` header, but the solution is valid for any header. Also, the example was showing the use of the attribute to solve the versioning problem, but this can be easily extended to any situation that requires handling requests in a different way, for the same HTTP verb and path.

The example used in this post is not a suggestion that this is the right way to handle API versions. There are certainly different options for that, and the right one would depend on your particular situation. The example was just intended to ilustrate how to route requests based on different header values when the HTTP verb and path are the same.

Keep dotnetting...
