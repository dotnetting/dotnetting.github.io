---
title: "Testing: Simulate database failures using EF Core Interceptors"
date: 2021-05-06
description: "This post shows how to simulate failure scenarios using Entity Framework Core Interceptors. This technique enables interception and modification of EF Core operations, including low-level database operations. This is compatible with SQL Server and other relational database providers."
summary: "The post explains how to use **EF Core Interceptors** to simulate database failures, enabling tests for application resiliency. It introduces a custom `MockFailCommandInterceptor` that selectively throws exceptions during specific database operations, such as `INSERT`, to test failure scenarios. This approach ensures that failure handling is thoroughly tested."
tags: ["ef core", "database", "interceptors", "csharp", ".net core", "sql server", "testing", "unit test", "resiliency"]
draft: false
---
One of the first things we learn about resiliency is that dependencies will fail at some point, and our applications must be prepared to deal gracefully with these failures. What _gracefully_ means will vary from app to app, but in all cases, this _graceful_ behaviour should be covered by some sort of tests. We have therefore to cover those failure scenarios to ensure our applications are resilient. In this post I will show you how to use **Interceptors** to simulate failures when using Entity Framework Core with relational databases like SQL Server. 

## Problem

Very often we need to test code that depends on database access. As with any other dependency, databases can fail, making our application to fail if those failure situations are not properly handled. One way to ensure our app handle those failure cases properly is writting some tests that simulate these failling conditions. Let's see one way of doing it using Entity Framework Core (EF Core). I will not show you here how to write resilient code, this is out of the scope of this post. I will show you though, how to  simulate database failures using EF Core so you can create tests that ensure your application is resilient.

## Solution

EF Core 3.0 introduced [Interceptors](https://docs.microsoft.com/en-us/ef/core/logging-events-diagnostics/interceptors) as a way to intercept, modify and suppress EF Core operations. This is particularly handy to simulate a failure when executing low-level database operations against data such as `SELECT`, `INSERT`, `UPDATE` and `DELETE`.

Let's imagine we have to implement a service to manage Movies. The service implementation would look like as

```csharp
public class MovieService
{
    private readonly MovieDbContext _dbContext;

    public MovieService(MovieDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public async Task AddMovie(Movie movie, CancellationToken token = default)
    {
        _dbContext.Movies.Add(movie);
        await _dbContext.SaveChangesAsync(token);
    }

    // Other methods
}
```

And the `MovieDbContext` can be something like this

```csharp
public class MovieDbContext : DbContext
{
    public MovieDbContext(DbContextOptions<MovieDbContext> options) : base(options) { }
    public DbSet<Movie> Movies { get; set; }
}

public class Movie
{
    public int Id { get; set; }
    public string Title { get; set; }
    public int Year { get; set; }
}
```

Now, we want to add some tests to cover the class `MovieService`, including the scenarios when the underlying database fails. Let's first introduce a simple test, and the boilerplate code to create an instance of `MovieDbContext` for our tests. I'm using SQLite in-memory for my Unit Tests following the [best practices recommended by Microsoft](https://docs.microsoft.com/en-us/ef/core/testing/#approach-2-sqlite). It is fast, convenient and it is relational. Testing against the same database system your code will run in Production would be the preferred option, but not always possible. This is where SQLite presents itself as a good compromise. It is important you understand the tradeoffs when using it, though.

```csharp
public class MovieServiceTests : IDisposable
{
    private readonly DbConnection _keepAliveConnection;

    public MovieServiceTests()
    {
        _keepAliveConnection = new SqliteConnection("DataSource=:memory:");
        _keepAliveConnection.Open();
    }

    [Fact]
    public async Task AddMovieSucceeds()
    {
        var dbContext = CreateMovieDbContext(_keepAliveConnection);
        var sut = new MovieService(dbContext);

        await sut.AddMovie(new Movie
        {
            Title = "Home Alone",
            Year = 1990
        });

        var movie = await dbContext.Movies.SingleAsync();
        Assert.Equal("Home Alone", movie.Title);
    }

    // Other tests

    public void Dispose() => _keepAliveConnection.Close();

    public static MovieDbContext CreateMovieDbContext(DbConnection connection)
    {
        var optionsBuilder = new DbContextOptionsBuilder<MovieDbContext>()
            .UseSqlite(connection);

        var dbContext = new MovieDbContext(optionsBuilder.Options);
        dbContext.Database.EnsureCreated();

        return dbContext;
    }
}
```

With those testing foundations in place, now we can write some tests to cover the scenario when the underlying database fails. One easy way of doing this is extending from the class [`DbCommandInterceptor`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.dbcommandinterceptor) that already implements the interface [`IInterceptor`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.diagnostics.iinterceptor), and just override the relevant method to force the desired error.

Let's say we want to test the case when an error occurs at the database when adding a new movie. In that case, we could throw an exception just before executing the `INSERT` command. One way to achieve this would be as follows.

```csharp
public class MockFailCommandInterceptor : DbCommandInterceptor
{
    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(DbCommand command, CommandEventData eventData, InterceptionResult<DbDataReader> result,
        CancellationToken cancellationToken = new CancellationToken())
    {
        if (command.CommandText.StartsWith("INSERT"))
        {
            // Throw to simulate a database failure, that will cause EF Core to throw DbUpdateException
            throw new Exception();
        }

        return base.ReaderExecutingAsync(command, eventData, result, cancellationToken);
    }
}
```

Now we have an interceptor that will throw an exception every time a command that starts with _INSERT_ is about to be passed to the underlying database. Because we want this failure just in some tests, not all, we have to selectively decide whether or not to use this interceptor. For that, we will modify the method that creates the `MovieDbContext` to receive the interceptor.

```csharp
public static MovieDbContext CreateMovieDbContext(DbConnection connection, params IInterceptor[] interceptors)
{
    var optionsBuilder = new DbContextOptionsBuilder<MovieDbContext>()
        .UseSqlite(connection);

    if (interceptors != null)
    {
        // Add interceptors to modify behaviour (i.e. simulate failures)
        optionsBuilder.AddInterceptors(interceptors);
    }

    var dbContext = new MovieDbContext(optionsBuilder.Options);
    dbContext.Database.EnsureCreated();

    return dbContext;
}
```

Now, if we don't pass any interceptors to this method, like in the previous test `AddMovieSucceeds`, EF Core will hand the queries directly to the provider; normal behavior. That would allow us to test all the cases when the database doesn't fail. But this also allows us to inject an instance of `MockFailCommandInterceptor` when we want to test a database failure.

```csharp
[Fact]
public async Task AddMovieFails()
{
    var dbContext = CreateMovieDbContext(_keepAliveConnection, new MockFailCommandInterceptor());
    var sut = new MovieService(dbContext);

    await Assert.ThrowsAsync<DbUpdateException>(() => 
        sut.AddMovie(new Movie
        {
            Title = "Home Alone",
            Year = 1990
        }));
}
```

We can make the `MockFailCommandInterceptor` class more versatile having a constructor that receives a predicate `Func<DbCommand, bool>` to decide when to throw, and also the actual exception to throw.

```csharp
public class MockFailCommandInterceptor : DbCommandInterceptor
{
    private readonly Func<DbCommand, bool> _predicate;
    private readonly Exception _ex;

    public MockFailCommandInterceptor(Func<DbCommand, bool> predicate, Exception exception)
    {
        _predicate = predicate;
        _ex = exception;
    }
    public override ValueTask<InterceptionResult<DbDataReader>> ReaderExecutingAsync(DbCommand command, CommandEventData eventData, InterceptionResult<DbDataReader> result, CancellationToken cancellationToken = new CancellationToken())
    {
        if (_predicate(command)) { throw _ex; }
        return base.ReaderExecutingAsync(command, eventData, result, cancellationToken);
    }
}
```

We can now use the _upgraded_ `MockFailCommandInterceptor` in our unit test as follows

```csharp
[Fact]
public async Task AddMovieFails()
{
    var dbContext = CreateMovieDbContext(_keepAliveConnection,
        new MockFailCommandInterceptor(
            command => command.CommandText.StartsWith("INSERT"),
            new Exception()));

    var sut = new MovieService(dbContext);

    await Assert.ThrowsAsync<DbUpdateException>(() => 
        sut.AddMovie(new Movie
        {
            Title = "Home Alone",
            Year = 1990
        }));
}
```

## Final thoughts

EF Core Interceptors are not something you would use regularly in your Production code. However, for testing failure scenarios, Interpectors are a nice and easy way to simulate database errors. Note that this is compatible only with relational database providers such as SQL Server. Also note that writing tests to cover failure scenarios alone doesn't guarantee that your application is resilient, but not covering them guarantees unpredictable, generaly _ungraceful_, behaviour when a dependency fails. 

Keep _gracefully_ dotnetting...