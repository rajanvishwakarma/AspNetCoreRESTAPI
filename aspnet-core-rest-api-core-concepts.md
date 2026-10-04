# ASP.NET Core REST APIs: the core concepts, in dependency order

A question-and-answer study guide. Each concept builds only on concepts that come before it, and each one ends with a single-file program you can run and a set of requests whose output you can compare with mine.

| | |
|---|---|
| **Target** | ASP.NET Core 10 on .NET 10 (the current long-term-support release) |
| **Tested with** | .NET SDK 10.0.112, ASP.NET Core runtime 10.0.12, Ubuntu 24.04, on 4 October 2026 |
| **What "tested" means** | Sections 1–26: I compiled and ran every example and sent every request shown. The output blocks are what came back. |
| **What was not run** | Sections 27–30 (JWT bearer, EF Core, OpenAPI, integration testing) need NuGet packages, and nuget.org was not reachable from the machine I tested on. Their code follows Microsoft's documentation but I did not execute it. Each of those sections says so again at the top. |

---

## How to use this guide

### What you need

- The **.NET 10 SDK** (`dotnet --version` should print `10.0.x`).
- **curl**, for sending requests.
- Two terminals: one runs the example, the other sends requests.

### Running an example

Every example is one C# file. .NET 10 can run a single file directly, with no project file:

```bash
# terminal 1
dotnet run 03-di.cs

# terminal 2
curl -s http://localhost:5000/lifetimes
```

Save each code block under the file name given above it. Stop an example with Ctrl+C before starting the next one, because they all listen on `http://localhost:5000`.

Every example starts with the same two lines:

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false
```

- `#:sdk Microsoft.NET.Sdk.Web` makes the file a web app: it references ASP.NET Core and adds the usual implicit `using` directives.
- `#:property PublishAot=false` is required. Single-file apps turn native AOT on by default, and that switches off reflection-based JSON serialization. When I ran an example without this line, the build printed `IL2026`/`IL3050`/`RDG004` warnings and the first request that returned JSON failed with HTTP 500 (`NotSupportedException: JsonTypeInfo metadata for type ... was not provided`).

### Running in a normal project instead

Use this if you prefer Visual Studio, or want a `.csproj`:

```bash
dotnet new web -o Demo
cd Demo
# replace Program.cs with the example, then DELETE the two lines that start with "#:"
dotnet run --no-launch-profile
```

- The `#:` lines must go. Left in, the project build fails with `error CS9298: '#:' directives can be only used in file-based programs`.
- `--no-launch-profile` makes the project behave like the single-file run: port 5000 and the Production environment. Without it, the template's `launchSettings.json` picks a different port (5045 on my machine) and sets the Development environment, so some output will differ from what is shown here.

I ran one example (section 9) this way to confirm the steps work.

### Reading the output blocks

- Lines starting with `# terminal 1:` show how the app was started. Lines starting with `$` are the commands I ran in the second terminal. Everything else is output.
- I removed three response headers from every block because they add nothing: `Date`, `Server: Kestrel` and `Transfer-Encoding: chunked`. Your output will include them.
- `traceId` values, request ids and timings are different on every run.
- Commands use bash syntax. On Windows, run them in Git Bash or WSL. In PowerShell, call `curl.exe` (plain `curl` is an alias for a different command in Windows PowerShell), set environment variables with `$env:NAME = "value"`, and expect to adjust the quoting around JSON bodies; putting the JSON in a file and sending it with `-d "@body.json"` avoids the quoting problem.

### One thing to know before you start: the environment

A single-file run starts in the **Production** environment, because nothing sets `ASPNETCORE_ENVIRONMENT`. That is deliberate here: you see what a deployed API does, such as empty error bodies and no exception details. Where Development behaves differently, the text says so.

### Older .NET versions

I tested on .NET 10 only. Two things are certain to need .NET 10: running a `.cs` file directly, and `AddValidation()` in section 13. Many of the other APIs arrived in .NET 7 or .NET 8, and I name the release for some of them, but I did not run anything on .NET 8 or 9. Treat this as a .NET 10 guide.

---

## Concept map

Read top to bottom. "Builds on" lists the sections a concept needs.

| # | Concept | Builds on |
|---|---|---|
| **Part 1: Foundations** | | |
| 1 | HTTP and REST | none |
| 2 | The host: `Program.cs`, Kestrel, environments | 1 |
| 3 | Dependency injection and service lifetimes | 2 |
| 4 | Configuration and the options pattern | 2, 3 |
| 5 | Logging | 3, 4 |
| **Part 2: The request pipeline** | | |
| 6 | Middleware | 2, 3 |
| 7 | Routing and endpoints | 6 |
| **Part 3: Building endpoints** | | |
| 8 | Minimal API handlers and parameter binding | 3, 7 |
| 9 | Responses: results and status codes | 1, 8 |
| 10 | JSON serialization and DTOs | 8, 9 |
| 11 | A complete CRUD resource with route groups | 3, 8, 9, 10 |
| 12 | Controllers | 3, 7, 9, 10 |
| **Part 4: Making it robust** | | |
| 13 | Validation | 8, 12 |
| 14 | Error handling and problem details | 6, 9 |
| 15 | Filters | 6, 8, 12 |
| 16 | Async, cancellation and timeouts | 8 |
| **Part 5: Security and traffic control** | | |
| 17 | Authentication | 3, 6 |
| 18 | Authorization | 17 |
| 19 | CORS | 6, 7 |
| 20 | Rate limiting | 6, 7 |
| **Part 6: Performance and integration** | | |
| 21 | Caching | 6, 9 |
| 22 | Calling other services with `IHttpClientFactory` | 3, 16 |
| 23 | Background services | 3, 16 |
| **Part 7: Operating the API** | | |
| 24 | Health checks | 3, 7 |
| 25 | API design: paging, versioning, concurrency | 9, 11, 21 |
| 26 | Hosting: reverse proxies, HTTPS, server limits | 4, 6 |
| **Part 8: Needs NuGet packages (not executed by me)** | | |
| 27 | JWT bearer authentication | 17, 18 |
| 28 | Data access with EF Core | 3, 11, 16 |
| 29 | OpenAPI documents | 9, 12 |
| 30 | Integration testing | 2, 3 |

---

# Part 1: Foundations

## 1. HTTP and REST

**Builds on:** nothing. Everything else in this guide is a way of reading an HTTP request or writing an HTTP response.

**Q 1.1. What is in an HTTP request and an HTTP response?**

A request has a **method** (`GET`, `POST`, ...), a **target** (path plus optional query string), **headers** (name/value metadata such as `Accept` and `Content-Type`) and an optional **body**. A response has a **status code**, **headers** and an optional **body**. ASP.NET Core hands you both as one object, `HttpContext`, with `context.Request` and `context.Response`.

**Q 1.2. What makes an API "RESTful" in practice?**

- **Resources have URLs.** `/orders/42` names a thing. URLs are nouns, not verbs.
- **The HTTP method says what to do** with the resource.
- **The status code says what happened.** Clients should not have to parse a body to learn whether a call worked.
- **Requests are stateless.** Each request carries everything needed to process it (for example its credentials). The server keeps no per-client conversation state between requests.
- **Representations are negotiated with headers.** `Content-Type` describes the body being sent, `Accept` says what the client wants back. For most APIs both are `application/json`.

**Q 1.3. What do the methods mean, and what are "safe" and "idempotent"?**

| Method | Typical use | Safe | Idempotent |
|---|---|---|---|
| GET | Read a resource or a list | yes | yes |
| POST | Create a resource, or trigger an action | no | no |
| PUT | Replace a resource completely | no | yes |
| PATCH | Change part of a resource | no | not necessarily |
| DELETE | Remove a resource | no | yes |

**Safe** means the request does not change server state. **Idempotent** means that sending the same request several times leaves the server in the same state as sending it once. This matters because clients, proxies and retry policies resend requests after timeouts. Resending a GET, PUT or DELETE is harmless. Resending a POST may create a duplicate, which is why section 25 discusses idempotency keys.

**Q 1.4. Which status codes must I know?**

| Code | Meaning | Where it appears in this guide |
|---|---|---|
| 200 OK | Success with a body | everywhere |
| 201 Created | A resource was created; `Location` header points to it | 9, 11, 12 |
| 202 Accepted | Work was queued, not finished | 9, 23 |
| 204 No Content | Success, nothing to return | 9, 11 |
| 304 Not Modified | The client's cached copy is still valid | 21 |
| 400 Bad Request | Malformed or invalid input | 8, 13 |
| 401 Unauthorized | Not authenticated (no or bad credentials) | 17, 18 |
| 403 Forbidden | Authenticated, but not allowed | 18 |
| 404 Not Found | No such resource or route | 7, 9 |
| 405 Method Not Allowed | The path exists, but not for this method | 7 |
| 409 Conflict | The request conflicts with current state | 9, 14 |
| 412 Precondition Failed | An `If-Match` condition was false | 25 |
| 413 Content Too Large | The request body exceeds the server limit | 26 |
| 415 Unsupported Media Type | The body's `Content-Type` is not accepted | 8 |
| 428 Precondition Required | The server insists on a conditional request | 25 |
| 429 Too Many Requests | Rate limit exceeded | 20 |
| 500 Internal Server Error | Unhandled failure on the server | 14 |
| 503 Service Unavailable | Temporarily unable to serve | 24 |
| 504 Gateway Timeout | A time limit was exceeded | 16 |

As a rule: 2xx means it worked, 4xx means the client must change the request, 5xx means the server failed and the same request might work later.

### Example: `01-http-anatomy.cs`

This program has no routing and no JSON. It shows the raw request and builds the response by hand.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var app = WebApplication.CreateBuilder(args).Build();

// app.Run(delegate) registers ONE terminal handler that answers every request,
// whatever its method or path. Nothing here knows about "routes" yet.
app.Run(async context =>
{
    HttpRequest req = context.Request;

    using var reader = new StreamReader(req.Body);
    string body = await reader.ReadToEndAsync();

    // The three parts of an HTTP response: status code, headers, body.
    context.Response.StatusCode = req.Method == "POST" ? 201 : 200;
    context.Response.Headers["X-Handled-By"] = "raw-delegate";
    context.Response.ContentType = "text/plain";

    await context.Response.WriteAsync(
        $"Method       : {req.Method}\n" +
        $"Path         : {req.Path}\n" +
        $"Query string : {req.QueryString}\n" +
        $"Accept       : {req.Headers.Accept}\n" +
        $"Content-Type : {req.ContentType}\n" +
        $"Body         : {body}\n");
});

app.Run();   // no arguments: start the web server and block until Ctrl+C
```

### Run it and verify

```text
# terminal 1:  dotnet run 01-http-anatomy.cs

$ curl -si "http://localhost:5000/orders/42?expand=items" -H "Accept: application/json"
HTTP/1.1 200 OK
Content-Type: text/plain
X-Handled-By: raw-delegate

Method       : GET
Path         : /orders/42
Query string : ?expand=items
Accept       : application/json
Content-Type : 
Body         : 

$ curl -si -X POST http://localhost:5000/orders -H "Content-Type: application/json" -d '{"sku":"A1"}'
HTTP/1.1 201 Created
Content-Type: text/plain
X-Handled-By: raw-delegate

Method       : POST
Path         : /orders
Query string : 
Accept       : */*
Content-Type : application/json
Body         : {"sku":"A1"}
```

**What to notice**

- The same delegate answered `GET /orders/42` and `POST /orders`. Without routing, the path is just a string.
- curl sent `Accept: */*` when not told otherwise.
- The code set the status code (201 for POST), a custom header and the content type. Those are the three parts of a response.
- `app.Run(...)` appears twice with different meanings: with a delegate it adds a terminal request handler, without arguments it starts the server.

---

## 2. The host: `Program.cs`, Kestrel and environments

**Builds on:** 1.

**Q 2.1. What does `Program.cs` do?**

It has two phases, separated by `builder.Build()`:

1. **Builder phase.** `WebApplication.CreateBuilder(args)` prepares configuration, logging, the dependency injection container and the web server. You add to them through `builder.Services`, `builder.Configuration`, `builder.Logging` and `builder.Environment`.
2. **App phase.** `builder.Build()` returns a `WebApplication`. On it you add middleware (`app.Use...`) and endpoints (`app.Map...`), then call `app.Run()`.

After `Build()` the service registrations are frozen. The example proves it: trying to register a service afterwards throws `The service collection cannot be modified because it is read-only.`

**Q 2.2. What is Kestrel, and which address does it listen on?**

Kestrel is the cross-platform web server built into ASP.NET Core. It runs inside your process. When nothing configures an address it listens on `http://localhost:5000`, which is what the output below shows. You can change the address with the `--urls` command-line argument, the `ASPNETCORE_URLS` environment variable, or Kestrel configuration. A project created from a template also has `Properties/launchSettings.json`, which sets a port for local runs only; it is not used in deployment.

**Q 2.3. What is "the environment", and what does it change?**

The environment is a name, read from `ASPNETCORE_ENVIRONMENT` (or `DOTNET_ENVIRONMENT`). If the variable is not set, the name is `Production`. By convention the other names are `Development` and `Staging`. Code can test it with `app.Environment.IsDevelopment()`.

`Development` changes framework behaviour in ways you will meet later:

- Unhandled exceptions produce a detailed developer error page instead of an empty 500 (section 14).
- The DI container checks for lifetime mistakes at startup (section 3).
- `appsettings.Development.json` and user secrets are loaded (section 4).

Never run a public deployment in `Development`: the error page exposes stack traces and request details.

**Q 2.4. What happens when the app shuts down?**

Ctrl+C or a termination signal starts a graceful shutdown: `ApplicationStopping` fires, the server stops accepting new requests, and in-flight requests and hosted services get time to finish. The default limit is 30 seconds (`HostOptions.ShutdownTimeout`).

### Example: `02-host.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

// -- Phase 1: the BUILDER. Register services, read configuration, inspect the environment.
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<Greeter>();

// -- Build() freezes the service registrations and creates the application.
var app = builder.Build();

// -- Phase 2: the APP. Add middleware and endpoints, then run.
app.Lifetime.ApplicationStarted.Register(() => Console.WriteLine(">> ApplicationStarted"));
app.Lifetime.ApplicationStopping.Register(() => Console.WriteLine(">> ApplicationStopping"));

app.MapGet("/", (Greeter greeter) => greeter.Hello());

app.MapGet("/info", (IHostEnvironment env) => new
{
    env.ApplicationName,
    env.EnvironmentName,
    IsDevelopment = env.IsDevelopment(),
    Urls = app.Urls,
});

// Proof that registration is closed once Build() has run.
app.MapGet("/late-registration", () =>
{
    try
    {
        builder.Services.AddSingleton<object>();
        return "registered";
    }
    catch (InvalidOperationException ex)
    {
        return ex.Message;
    }
});

app.Run();   // start Kestrel and block until shutdown

class Greeter
{
    public string Hello() => "Hello from Kestrel";
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 02-host.cs

$ curl -s http://localhost:5000/
Hello from Kestrel

$ curl -s http://localhost:5000/info
{"applicationName":"02-host","environmentName":"Production","isDevelopment":false,"urls":["http://localhost:5000"]}

$ curl -s http://localhost:5000/late-registration
The service collection cannot be modified because it is read-only.

--- terminal 1: selected console lines from that run, including shutdown after Ctrl+C ---
      Now listening on: http://localhost:5000
      Hosting environment: Production
>> ApplicationStarted
>> ApplicationStopping
      Application is shutting down...

# terminal 1:  ASPNETCORE_ENVIRONMENT=Development dotnet run 02-host.cs

$ curl -s http://localhost:5000/info
{"applicationName":"02-host","environmentName":"Development","isDevelopment":true,"urls":["http://localhost:5000"]}
```

**What to notice**

- The console logger writes on a background thread, so the framework's lines and the `>>` lines can appear in a different order from run to run.
- With no environment variable the app reports `Production`. With `ASPNETCORE_ENVIRONMENT=Development` it reports `Development`. Nothing in the code changed.
- An anonymous object returned from a handler was written as JSON with camelCase property names (section 10).
- `Greeter` was never constructed by our code. The framework created it and passed it to the handler. That is dependency injection, the next section.

---

## 3. Dependency injection and service lifetimes

**Builds on:** 2.

**Q 3.1. What is dependency injection (DI), and why does ASP.NET Core depend on it?**

A class that needs another object (a repository, a logger, a clock) declares it as a constructor or handler parameter instead of creating it. A **container** knows how to build every registered type and supplies the parameters. You register types on `builder.Services`; the framework resolves them when it creates handlers, controllers and middleware.

The benefits are practical: classes depend on interfaces, so implementations can be swapped (for example in tests); object creation and clean-up are in one place; and every framework feature (logging, configuration, HTTP clients, authentication) is delivered the same way.

**Q 3.2. What are the three lifetimes?**

| Lifetime | Instances created | Use for |
|---|---|---|
| **Transient** | A new one every time it is requested | Lightweight, stateless helpers |
| **Scoped** | One per scope. ASP.NET Core creates a scope for each HTTP request | Per-request state, such as a database session or the current user |
| **Singleton** | One for the whole application | Shared, thread-safe state and caches |

The first two requests in the output below show this exactly: the transient ids are different even inside one request, the scoped id is shared inside a request and changes for the next request, and the singleton id never changes.

**Q 3.3. What is a "captive dependency"?**

A long-lived service holding a shorter-lived one. If a singleton takes a scoped service in its constructor, that scoped instance is kept for the life of the application: it is shared by every request and never released. With something like a database session this causes stale data and concurrency bugs.

The container can detect this, but only checks by default in the Development environment. I confirmed both behaviours with a singleton `Cache` that takes a scoped `Repo`:

- In Development the app refused to start: `Cannot consume scoped service 'Repo' from singleton 'Cache'.`
- In Production the same app started and served requests with the bug in place.

The example turns the checks on explicitly (`ValidateScopes`, `ValidateOnBuild`) so that it behaves the same in every environment. It also shows the related mistake of resolving a scoped service from the root container, which fails with `Cannot resolve scoped service 'ScopedService' from root provider.`

**Q 3.4. How does a singleton or background task use a scoped service correctly?**

It creates its own scope with `IServiceScopeFactory`, resolves the service from that scope, uses it, and disposes the scope. The `/manual-scope` endpoint shows the pattern; section 23 uses it for real.

**Q 3.5. How do services reach my code?**

- **Constructor parameters** of classes the container creates (services, controllers, middleware). C# primary constructors keep this short: `class OrderService(IClock clock)`.
- **Handler parameters** of minimal API endpoints, as in every example here.
- **Keyed services** (.NET 8+) when several implementations of one interface are registered: `[FromKeyedServices("sms")] INotifier notifier`.

Avoid calling `GetRequiredService` from application code as a general habit ("service locator"). It hides what a class depends on. It is fine in composition code such as `Program.cs` and inside a manually created scope.

**Q 3.6. Which lifetimes do the framework's own services have?**

I printed these from the container:

| Service | Lifetime |
|---|---|
| `ILogger<T>`, `IConfiguration`, `IOptions<T>`, `IOptionsMonitor<T>` | Singleton |
| `IOptionsSnapshot<T>` | Scoped |
| `IHttpClientFactory` | Singleton |
| Typed HTTP clients (`AddHttpClient<T>`) | Transient |
| Hosted services (`AddHostedService<T>`) | Singleton |
| Exception handlers (`AddExceptionHandler<T>`) | Singleton |
| `IHttpContextAccessor` | Singleton |

### Example: `03-di.cs`

Each service numbers its instances, so the output shows how many were created.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var builder = WebApplication.CreateBuilder(args);

// Both checks are ON by default only in the Development environment.
// Turning them on explicitly makes this example behave the same in every environment.
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;    // catch scoped services resolved from the root container
    options.ValidateOnBuild = true;   // check at startup that every registration can be constructed
});

builder.Services.AddTransient<TransientService>();   // a new instance every time one is requested
builder.Services.AddScoped<ScopedService>();         // one instance per scope (= per HTTP request)
builder.Services.AddSingleton<SingletonService>();   // one instance for the whole application

// Register an interface with its implementation: consumers depend on IClock only.
builder.Services.AddSingleton<IClock, FixedClock>();

// Keyed services (.NET 8+): several implementations of one interface, picked by key.
builder.Services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
builder.Services.AddKeyedSingleton<INotifier, SmsNotifier>("sms");

var app = builder.Build();

// Each parameter is resolved from the container. Asking twice shows which lifetimes share.
app.MapGet("/lifetimes", (TransientService t1, TransientService t2,
                          ScopedService s1, ScopedService s2,
                          SingletonService single) => new
{
    transient = new[] { t1.Id, t2.Id },
    scoped = new[] { s1.Id, s2.Id },
    singleton = single.Id,
});

app.MapGet("/clock", (IClock clock) => clock.Today.ToString("yyyy-MM-dd"));

app.MapGet("/notify", ([FromKeyedServices("sms")] INotifier notifier) => notifier.Send("hi"));

// Resolving a scoped service from the ROOT provider is a bug: it would never be released.
app.MapGet("/from-root", () =>
{
    try
    {
        return app.Services.GetRequiredService<ScopedService>().Id;
    }
    catch (InvalidOperationException ex)
    {
        return ex.Message;
    }
});

// The correct way to use a scoped service outside a request: create a scope yourself.
app.MapGet("/manual-scope", (IServiceScopeFactory scopeFactory) =>
{
    using IServiceScope scope = scopeFactory.CreateScope();
    return scope.ServiceProvider.GetRequiredService<ScopedService>().Id;
});

app.Run();

class TransientService
{
    static int _count;
    public string Id { get; } = $"transient#{Interlocked.Increment(ref _count)}";
}

class ScopedService
{
    static int _count;
    public string Id { get; } = $"scoped#{Interlocked.Increment(ref _count)}";
}

class SingletonService
{
    static int _count;
    public string Id { get; } = $"singleton#{Interlocked.Increment(ref _count)}";
}

interface IClock { DateOnly Today { get; } }
class FixedClock : IClock { public DateOnly Today => new(2026, 1, 15); }

interface INotifier { string Send(string message); }
class EmailNotifier : INotifier { public string Send(string message) => $"email: {message}"; }
class SmsNotifier : INotifier { public string Send(string message) => $"sms: {message}"; }
```

### Run it and verify

```text
# terminal 1:  dotnet run 03-di.cs

$ curl -s http://localhost:5000/lifetimes
{"transient":["transient#1","transient#2"],"scoped":["scoped#1","scoped#1"],"singleton":"singleton#1"}

$ curl -s http://localhost:5000/lifetimes
{"transient":["transient#3","transient#4"],"scoped":["scoped#2","scoped#2"],"singleton":"singleton#1"}

$ curl -s http://localhost:5000/clock
2026-01-15

$ curl -s http://localhost:5000/notify
sms: hi

$ curl -s http://localhost:5000/from-root
Cannot resolve scoped service 'ScopedService' from root provider.

$ curl -s http://localhost:5000/manual-scope
scoped#3
```

**What to notice**

- First request: `transient#1` and `transient#2` (two instances), `scoped#1` twice (one instance), `singleton#1`.
- Second request: new transients, a new scoped instance, the same singleton.
- `/manual-scope` returned `scoped#3`: a third scoped instance, created in the scope the endpoint opened itself.

---

## 4. Configuration and the options pattern

**Builds on:** 2, 3.

**Q 4.1. Where does configuration come from, and which source wins?**

Configuration is a set of key/value pairs merged from several **providers**. When two providers have the same key, the one added later wins. Microsoft documents the default order, from highest to lowest priority, as:

1. Command-line arguments
2. Environment variables (those not prefixed `ASPNETCORE_` or `DOTNET_`)
3. User secrets, in the Development environment
4. `appsettings.{Environment}.json`
5. `appsettings.json`
6. Host configuration (the prefixed environment variables, among others)

The `/providers` endpoint in the example prints the real list in load order (lowest priority first), so you can check this against your own machine. On .NET 10 I also saw two optional files named after the application (`<AppName>.settings.json` and `<AppName>.settings.{Environment}.json`) loaded after the `appsettings` files, in both a single-file app and a `dotnet new web` project.

**Q 4.2. How are nested keys written?**

Keys are hierarchical, with `:` as the separator: `Payments:TimeoutSeconds`. In JSON that is a nested object. In an environment variable use a double underscore, `Payments__TimeoutSeconds`, because `:` is not valid in variable names on every platform. On the command line use `--Payments:RetryCount=5`.

**Q 4.3. What is the options pattern, and why not inject `IConfiguration` everywhere?**

The options pattern binds one configuration section to a typed class and registers it in DI. Code then asks for `IOptions<PaymentOptions>` and reads real properties with real types.

Compared with reading `config["Payments:TimeoutSeconds"]` in many places, this gives you one definition of the settings, compile-time checking of names, type conversion, validation, and classes that are easy to test because they depend on a small options object and not on the whole configuration system.

**Q 4.4. What is the difference between `IOptions<T>`, `IOptionsSnapshot<T>` and `IOptionsMonitor<T>`?**

| Interface | Lifetime | Sees configuration changes after startup? |
|---|---|---|
| `IOptions<T>` | Singleton | No. The value is computed once. |
| `IOptionsSnapshot<T>` | Scoped | Yes. Recomputed for each request, constant within it. Cannot be injected into a singleton. |
| `IOptionsMonitor<T>` | Singleton | Yes. `CurrentValue` is always the latest, and `OnChange` notifies you. |

Use `IOptions<T>` unless you need reloading.

**Q 4.5. How do I stop the app starting with bad configuration?**

Validate the options and call `ValidateOnStart()`. Without it, validation runs only when the options are first used, which may be hours after deployment. The third run below sets an out-of-range value: the process fails immediately with a message naming the setting.

**Q 4.6. Where do secrets go?**

Not in `appsettings.json` and not in source control. In development use the Secret Manager (`dotnet user-secrets`), which keeps values outside the project folder. In production supply them through environment variables or a secret store such as a key vault.

### Example: `04-config-options.cs`

The example stays in one file by putting its "appsettings" values in an in-memory provider at the lowest priority.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.ComponentModel.DataAnnotations;
using Microsoft.Extensions.Configuration.Memory;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// Stand-in for appsettings.json so the example stays in one file.
// Inserted at index 0 = lowest priority, so environment variables and command-line
// arguments override it: the same relationship appsettings.json has with them.
builder.Configuration.Sources.Insert(0, new MemoryConfigurationSource
{
    InitialData = new Dictionary<string, string?>
    {
        ["Payments:BaseUrl"] = "https://sandbox.example.com",
        ["Payments:TimeoutSeconds"] = "30",
        ["Payments:RetryCount"] = "3",
    }
});

// Options pattern: bind a configuration section to a typed class, validate it, expose it via DI.
builder.Services.AddOptions<PaymentOptions>()
    .BindConfiguration("Payments")
    .ValidateDataAnnotations()   // honour [Required], [Range], ...
    .ValidateOnStart();          // fail when the app starts, not on first use

var app = builder.Build();

// Raw access: keys are strings, ':' separates levels, values are strings.
app.MapGet("/raw", (IConfiguration config) => new
{
    viaIndexer = config["Payments:TimeoutSeconds"],
    viaGetValue = config.GetValue<int>("Payments:TimeoutSeconds"),
    missingKey = config["Payments:DoesNotExist"],
});

// Typed access: what application code should normally use.
app.MapGet("/options", (IOptions<PaymentOptions> options) => options.Value);

// The providers in priority order: a later provider overrides an earlier one.
app.MapGet("/providers", (IConfiguration config) =>
    ((IConfigurationRoot)config).Providers.Select(p => p.ToString()));

app.Run();

class PaymentOptions
{
    [Required, Url]
    public string BaseUrl { get; set; } = "";

    [Range(1, 120)]
    public int TimeoutSeconds { get; set; }

    [Range(0, 10)]
    public int RetryCount { get; set; }
}
```

### Run it and verify

Three runs: defaults, then an environment variable plus a command-line argument, then an invalid value.

```text
# terminal 1:  dotnet run 04-config-options.cs

$ curl -s http://localhost:5000/raw
{"viaIndexer":"30","viaGetValue":30,"missingKey":null}

$ curl -s http://localhost:5000/options
{"baseUrl":"https://sandbox.example.com","timeoutSeconds":30,"retryCount":3}

$ curl -s http://localhost:5000/providers
   1. MemoryConfigurationProvider
   2. MemoryConfigurationProvider
   3. EnvironmentVariablesConfigurationProvider Prefix: 'ASPNETCORE_'
   4. MemoryConfigurationProvider
   5. EnvironmentVariablesConfigurationProvider Prefix: 'DOTNET_'
   6. JsonConfigurationProvider for 'appsettings.json' (Optional)
   7. JsonConfigurationProvider for 'appsettings.Production.json' (Optional)
   8. JsonConfigurationProvider for '04-config-options.settings.json' (Optional)
   9. JsonConfigurationProvider for '04-config-options.settings.Production.json' (Optional)
  10. EnvironmentVariablesConfigurationProvider
  11. Microsoft.Extensions.Configuration.ChainedConfigurationProvider
(the JSON array is shown one item per line, numbered, for readability)

# terminal 1:  Payments__TimeoutSeconds=45 dotnet run 04-config-options.cs -- --Payments:RetryCount=5

$ curl -s http://localhost:5000/options
{"baseUrl":"https://sandbox.example.com","timeoutSeconds":45,"retryCount":5}

# terminal 1:  Payments__TimeoutSeconds=999 dotnet run 04-config-options.cs

Unhandled exception. Microsoft.Extensions.Options.OptionsValidationException: DataAnnotation validation failed for 'PaymentOptions' members: 'TimeoutSeconds' with the error: 'The field TimeoutSeconds must be between 1 and 120.'.
(the process exits; nothing is listening on port 5000)
```

**What to notice**

- `viaIndexer` is the string `"30"`; `GetValue<int>` and the options class give the number `30`.
- A missing key is `null`, not an error. Typos in key names fail silently, which is another reason to bind and validate.
- In the second run, the environment variable changed `timeoutSeconds` to 45 and the command-line argument changed `retryCount` to 5. `baseUrl` kept its lowest-priority value.
- The provider list is in load order. The un-prefixed `EnvironmentVariablesConfigurationProvider` comes after the JSON files, so it overrides them. No command-line provider is listed in the first run because no arguments were passed.
- The last entry, a chained provider, is not application configuration. When I listed its keys it held four web-host settings (`applicationName` and three hosting-startup keys).
- Host configuration really is the fallback: with the same key in `appsettings.json` and in an `ASPNETCORE_`-prefixed environment variable, the value from `appsettings.json` won when I tried it.

---

## 5. Logging

**Builds on:** 3, 4.

**Q 5.1. How do I write a log entry?**

Inject `ILogger<T>` and call `LogInformation`, `LogWarning`, `LogError` and so on. `T` becomes the entry's **category**, normally the class doing the logging. The levels, from least to most severe, are `Trace`, `Debug`, `Information`, `Warning`, `Error`, `Critical`. The default minimum level is `Information` (I read this from `LoggerFilterOptions.MinLevel`); entries below the minimum are discarded.

**Q 5.2. Why `LogInformation("Order {OrderId}", id)` and not `LogInformation($"Order {id}")`?**

The first form is a **message template**. The logger keeps `OrderId` as a named field next to the rendered text. The output below shows it: `"State": { "OrderId": 42, "Customer": "acme", ... }`. A log system can then find every entry where `OrderId` is 42 without searching text. String interpolation throws that structure away, and it also builds the string even when the level is disabled.

**Q 5.3. How do I control what gets logged?**

By category prefix and level. The usual place is configuration:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

The example does the same in code with `SetMinimumLevel` and `AddFilter`. The most specific matching category prefix wins.

**Q 5.4. What are scopes?**

A scope attaches data to every entry written while it is open. ASP.NET Core opens scopes for each request, which is why the entries below carry a `TraceId`, a `ConnectionId` and the `RequestPath` without the code asking for them. `BeginScope` adds your own, for example a correlation or tenant id.

**Q 5.5. What is the `[LoggerMessage]` attribute for?**

It generates the logging method at compile time. The result avoids boxing and template parsing at run time and gives each message a fixed event id and level. Use it on hot paths and for messages you want to keep stable.

**Q 5.6. What must never be logged?**

Passwords, tokens, API keys, connection strings and personal data. Log entries are copied to many systems and kept for a long time. Section 14 shows a case where an exception message contains a password: it stays out of the HTTP response but it does reach the log, so the log itself needs protecting.

### Example: `05-logging.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();                                      // drop the default providers
builder.Logging.AddJsonConsole(o => o.IncludeScopes = true);           // one JSON object per entry
builder.Logging.SetMinimumLevel(LogLevel.Debug);                       // floor for every category
builder.Logging.AddFilter("Microsoft.AspNetCore", LogLevel.Warning);   // quieten framework categories

var app = builder.Build();

// ILogger<T>: T becomes the log CATEGORY (here "Program").
app.MapGet("/orders/{id:int}", (int id, ILogger<Program> logger) =>
{
    // A message TEMPLATE plus arguments. {OrderId} and {Customer} become named fields.
    logger.LogInformation("Fetching order {OrderId} for {Customer}", id, "acme");

    // A scope attaches the same data to every entry written inside it.
    using (logger.BeginScope("Correlation {CorrelationId}", "c-123"))
    {
        logger.LogDebug("Inside the scope");
    }

    logger.LogTrace("Trace is below the Debug minimum, so this entry is dropped");

    Log.OrderShipped(logger, id, parcels: 3);   // source-generated, allocation-free logging
    return Results.Ok(id);
});

app.Run();

static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Warning,
        Message = "Order {OrderId} shipped in {Parcels} parcels")]
    public static partial void OrderShipped(ILogger logger, int orderId, int parcels);
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 05-logging.cs

$ curl -s http://localhost:5000/orders/42
42

--- console output of the app (entries from the Program category) ---
{
  "EventId": 0,
  "LogLevel": "Information",
  "Category": "Program",
  "Message": "Fetching order 42 for acme",
  "State": {
    "OrderId": 42,
    "Customer": "acme",
    "{OriginalFormat}": "Fetching order {OrderId} for {Customer}"
  },
  "Scopes": [
    "SpanId:902ab36b78cf6928, TraceId:e53268e4d73d9d575c3b03ec4282add5, ParentId:0000000000000000",
    "ConnectionId:0HNP1TMBLTOCK",
    "RequestPath:/orders/42 RequestId:0HNP1TMBLTOCK:00000001"
  ]
}

{
  "EventId": 0,
  "LogLevel": "Debug",
  "Category": "Program",
  "Message": "Inside the scope",
  "State": {
    "{OriginalFormat}": "Inside the scope"
  },
  "Scopes": [
    "SpanId:902ab36b78cf6928, TraceId:e53268e4d73d9d575c3b03ec4282add5, ParentId:0000000000000000",
    "ConnectionId:0HNP1TMBLTOCK",
    "RequestPath:/orders/42 RequestId:0HNP1TMBLTOCK:00000001",
    "Correlation c-123"
  ]
}

{
  "EventId": 1001,
  "LogLevel": "Warning",
  "Category": "Program",
  "Message": "Order 42 shipped in 3 parcels",
  "State": {
    "OrderId": 42,
    "Parcels": 3,
    "{OriginalFormat}": "Order {OrderId} shipped in {Parcels} parcels"
  },
  "Scopes": [
    "SpanId:902ab36b78cf6928, TraceId:e53268e4d73d9d575c3b03ec4282add5, ParentId:0000000000000000",
    "ConnectionId:0HNP1TMBLTOCK",
    "RequestPath:/orders/42 RequestId:0HNP1TMBLTOCK:00000001"
  ]
}

(each entry is one line on the console; shown indented here, with every scope reduced to its Message text)
```

**What to notice**

- Three entries were written. The `LogTrace` call produced nothing because `Trace` is below the configured minimum of `Debug`.
- `OrderId` and `Parcels` are fields in `State`, not just text inside `Message`.
- Every entry has three scopes added by the framework. The `Inside the scope` entry has a fourth, `Correlation c-123`, from `BeginScope`.
- The source-generated entry has `EventId` 1001 and level `Warning`, as declared in the attribute.

---

# Part 2: The request pipeline

## 6. Middleware

**Builds on:** 2, 3.

**Q 6.1. What is middleware?**

A piece of code that sits in the path of every request. Each middleware receives the `HttpContext` and a `next` delegate. It can do work, call `next` to pass the request on, and do more work when `next` returns. Middleware components are chained in the order you add them, which gives a nested structure: the first one added is the outermost, so it runs first on the way in and last on the way out.

**Q 6.2. What is the difference between `Use`, `Run` and `Map`?**

- `app.Use(...)` adds middleware that may call `next`.
- `app.Run(...)` adds a **terminal** delegate. It never calls `next`, so nothing after it runs.
- `app.Map("/prefix", ...)` creates a **branch**: requests whose path starts with the prefix go down a separate pipeline. `app.UseWhen(predicate, ...)` branches on any condition and rejoins the main pipeline afterwards.

Do not confuse `app.Map("/prefix", branch => ...)` with `app.MapGet(...)`. The first is pipeline branching; the second defines an endpoint (section 7).

**Q 6.3. What does "short-circuiting" mean?**

A middleware that does not call `next` ends the request there. Authorization returning 401, a cache returning a stored response and a rate limiter returning 429 all work this way. In the example, `?block` makes middleware B stop the request; A's "after" code still runs because A is outside B.

**Q 6.4. Why does order matter so much?**

Because each middleware only affects what comes after it. An exception handler must be early, or it cannot catch exceptions from later middleware. Authentication must run before authorization, or there is no user to authorize. The appendix has a recommended order for an API.

**Q 6.5. What does `WebApplication` add without being asked?**

- The developer exception page, first in the pipeline, when the environment is Development.
- Routing (`UseRouting`) near the start and endpoint execution at the end, if you mapped endpoints and did not call `UseRouting` yourself.
- Authentication and authorization middleware, if their services are registered. I confirmed this by deleting the `UseAuthentication()` and `UseAuthorization()` calls from the section 17 example: it still returned 401 without a key and the user's data with one.

Calling them explicitly is still worthwhile, because it makes the order visible and lets you place other middleware between them.

**Q 6.6. Why do I get "Headers are read-only, response has already started"?**

Status and headers are sent with the first byte of the body. After that they cannot change. The example provokes the error deliberately and prints the message. Set headers before writing the body, or register a callback with `context.Response.OnStarting(...)`, which runs just before the headers are sent.

**Q 6.7. How do I write reusable middleware, and what about its dependencies?**

Write a class with a constructor that takes `RequestDelegate next` and a method `InvokeAsync(HttpContext context)`, and add it with `app.UseMiddleware<T>()`. Such a class is created **once** for the application. So constructor parameters must be singletons (or safe to hold forever). A scoped service goes in the `InvokeAsync` parameter list, where it is resolved per request.

### Example: `06-middleware.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Middleware runs in the order it is added. This class-based one is first, so it wraps everything.
app.UseMiddleware<RequestIdMiddleware>();

// Inline middleware A.
app.Use(async (context, next) =>
{
    context.Response.ContentType = "text/plain";
    await context.Response.WriteAsync("A: before\n");

    await next(context);   // hand the request to the rest of the pipeline

    await context.Response.WriteAsync("A: after\n");
    try
    {
        // Headers were sent with the first byte of the body, so this is too late.
        context.Response.Headers["X-Too-Late"] = "1";
    }
    catch (InvalidOperationException ex)
    {
        await context.Response.WriteAsync($"A: {ex.Message}\n");
    }
});

// Inline middleware B: short-circuits when ?block is present by NOT calling next.
app.Use(async (context, next) =>
{
    if (context.Request.Query.ContainsKey("block"))
    {
        await context.Response.WriteAsync("B: short-circuited, nothing after me runs\n");
        return;
    }

    await context.Response.WriteAsync("B: before\n");
    await next(context);
    await context.Response.WriteAsync("B: after\n");
});

// Map branches the pipeline for a path prefix. The branch ends in its own terminal delegate.
app.Map("/branch", branch =>
    branch.Run(async context => await context.Response.WriteAsync("branch: terminal\n")));

// The endpoint is the end of the main pipeline.
app.MapGet("/", async (HttpContext context) => await context.Response.WriteAsync("endpoint\n"));

app.Run();

// Convention-based middleware: a constructor taking RequestDelegate and an InvokeAsync method.
class RequestIdMiddleware(RequestDelegate next, ILogger<RequestIdMiddleware> logger)
{
    static int _counter;

    public async Task InvokeAsync(HttpContext context)
    {
        int id = Interlocked.Increment(ref _counter);
        context.Response.Headers["X-Request-Id"] = id.ToString();   // fine: nothing sent yet

        logger.LogInformation("Request {Id} {Path} starting", id, context.Request.Path);
        await next(context);
        logger.LogInformation("Request {Id} finished with {Status}", id, context.Response.StatusCode);
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 06-middleware.cs

$ curl -si http://localhost:5000/
HTTP/1.1 200 OK
Content-Type: text/plain
X-Request-Id: 1

A: before
B: before
endpoint
B: after
A: after
A: Headers are read-only, response has already started.

$ curl -s "http://localhost:5000/?block"
A: before
B: short-circuited, nothing after me runs
A: after
A: Headers are read-only, response has already started.

$ curl -s http://localhost:5000/branch
A: before
B: before
branch: terminal
B: after
A: after
A: Headers are read-only, response has already started.

--- console output of the app (RequestIdMiddleware lines) ---
info: RequestIdMiddleware[0]
      Request 1 / starting
info: RequestIdMiddleware[0]
      Request 1 finished with 200
info: RequestIdMiddleware[0]
      Request 2 / starting
info: RequestIdMiddleware[0]
      Request 2 finished with 200
info: RequestIdMiddleware[0]
      Request 3 /branch starting
info: RequestIdMiddleware[0]
      Request 3 finished with 200
```

**What to notice**

- The body reads `A: before`, `B: before`, `endpoint`, `B: after`, `A: after`: in order on the way in, reverse order on the way out.
- `X-Request-Id` was set successfully because it happened before any body was written. `X-Too-Late` was not, and the exception message is the last line of each body.
- With `?block`, neither `B: after` nor `endpoint` appears, but `A: after` does.
- `/branch` went through A and B, then into the branch. The endpoint for `/` was never involved.

---

## 7. Routing and endpoints

**Builds on:** 6.

**Q 7.1. What is an endpoint, and what does routing do?**

An **endpoint** is a piece of code plus the information needed to select it: a route template, the allowed HTTP methods, and metadata such as authorization requirements. **Routing** compares each request with all endpoints and selects one. It is done by two middleware: the routing middleware selects the endpoint, and the endpoint middleware at the end of the pipeline runs it. Middleware placed between the two can see which endpoint was selected and read its metadata. That is how authorization, CORS and rate limiting apply per-endpoint rules.

**Q 7.2. What can a route template contain?**

| Syntax | Meaning | Example |
|---|---|---|
| literal | Must match exactly (case-insensitive) | `/products/featured` |
| `{name}` | Captures one segment | `/products/{slug}` |
| `{name?}` | Optional segment | `/reports/{year}/{month?}` |
| `{name=value}` | Default when the segment is absent | `/{culture=en}/home` |
| `{**name}` | Catch-all: the rest of the path | `/files/{**path}` |
| `{name:constraint}` | Segment must satisfy a constraint | `{id:int}`, `{id:guid}`, `{age:range(18,120)}`, `{code:length(3)}`, `{name:alpha}`, `{slug:regex(^[a-z0-9-]+$)}` |

Constraints can be chained: `{year:int:min(2000)}`.

**Q 7.3. Several templates match the same URL. Which one wins?**

The most specific. A literal segment beats a constrained parameter, which beats an unconstrained parameter, which beats a catch-all. The example maps `/products/featured`, `/products/{id:int}` and `/products/{slug}`, and each of three URLs lands on a different one.

If two endpoints are equally specific, routing cannot choose. I mapped the same GET route twice: the request failed with HTTP 500 and the log showed `AmbiguousMatchException: The request matched multiple endpoints.`

**Q 7.4. Should I use route constraints to validate input?**

No. A failed constraint means "this route does not match", so the client gets **404**, not 400. In the example `/reports/2025/13` and `/reports/1999` both return 404. Use constraints to tell routes apart (`{id:int}` versus `{slug}`). Validate values in the handler or with validation attributes (section 13), where a failure produces a 400 with an explanation.

**Q 7.5. What is the difference between 404 and 405?**

404: no endpoint matches the path. 405: the path matches an endpoint, but not for this method. ASP.NET Core adds an `Allow` header listing the methods that would work. The last request below is a GET to a POST-only path.

**Q 7.6. How do I produce URLs to my own endpoints?**

Name the endpoint (`.WithName("GetProduct")`) and ask `LinkGenerator` for its path. If the route template later changes, generated links change with it. `Results.CreatedAtRoute` (section 9) uses the same mechanism for the `Location` header. Endpoint names must be unique in the application.

### Example: `07-routing.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var app = WebApplication.CreateBuilder(args).Build();

// Three templates that could all match "/products/<something>". Routing picks the most specific.
app.MapGet("/products/featured", () => "literal segment 'featured'");
app.MapGet("/products/{id:int}", (int id) => $"int constraint: id={id}").WithName("GetProduct");
app.MapGet("/products/{slug}", (string slug) => $"unconstrained: slug={slug}");

// Several constraints on one parameter, plus an optional parameter (the trailing ?).
app.MapGet("/reports/{year:int:min(2000)}/{month:int:range(1,12)?}",
    (int year, int? month) => $"year={year}, month={month?.ToString() ?? "(whole year)"}");

// Catch-all: ** captures the rest of the path, slashes included.
app.MapGet("/files/{**path}", (string path) => $"catch-all: {path}");

// Only POST is mapped for this path.
app.MapPost("/orders", () => "order created");

// LinkGenerator builds a URL from a route NAME, so URLs are never hard-coded.
app.MapGet("/link", (LinkGenerator links) => links.GetPathByName("GetProduct", new { id = 5 }));

app.Run();
```

### Run it and verify

```text
# terminal 1:  dotnet run 07-routing.cs

$ curl -s http://localhost:5000/products/featured
literal segment 'featured'

$ curl -s http://localhost:5000/products/42
int constraint: id=42

$ curl -s http://localhost:5000/products/blue-shirt
unconstrained: slug=blue-shirt

$ curl -s http://localhost:5000/reports/2025
year=2025, month=(whole year)

$ curl -s http://localhost:5000/reports/2025/7
year=2025, month=7

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/reports/2025/13
404

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/reports/1999
404

$ curl -s http://localhost:5000/files/docs/2026/readme.txt
catch-all: docs/2026/readme.txt

$ curl -s http://localhost:5000/link
/products/5

$ curl -si http://localhost:5000/orders
HTTP/1.1 405 Method Not Allowed
Content-Length: 0
Allow: POST
```

---

# Part 3: Building endpoints

## 8. Minimal API handlers and parameter binding

**Builds on:** 3, 7.

**Q 8.1. What is a minimal API endpoint?**

A route plus a delegate: `app.MapGet("/items/{id}", (int id) => ...)`. `MapGet`, `MapPost`, `MapPut`, `MapPatch` and `MapDelete` each register an endpoint for one HTTP method. The framework inspects the delegate's parameters and fills them in for each request. That process is **parameter binding**.

**Q 8.2. How does the framework decide where each parameter comes from?**

By explicit attribute if there is one, otherwise by these rules:

| Parameter | Source |
|---|---|
| Has `[FromRoute]`, `[FromQuery]`, `[FromHeader]`, `[FromBody]`, `[FromForm]`, `[FromServices]` | As stated |
| `HttpContext`, `HttpRequest`, `HttpResponse`, `ClaimsPrincipal`, `CancellationToken` | Supplied by the framework |
| Simple type (`int`, `string`, `Guid`, `DateTime`, enums, anything with `TryParse`) whose name is in the route template | Route value |
| Simple type whose name is not in the route template | Query string |
| Array of simple types | Repeated query string keys |
| A type registered in the DI container | The container |
| Anything else (a class or record) | The JSON request body |

Headers are never inferred. They always need `[FromHeader]`.

**Q 8.3. What makes a parameter required or optional?**

A parameter is optional if its type is nullable (`string?`, `int?`) or it has a default value. Otherwise it is required, and a request without it is rejected with **400** before your handler runs. In Production the 400 has no body. In Development I got the reason as text: `BadHttpRequestException: Required parameter "string Term" was not provided from query string.`

**Q 8.4. What are the rules for the request body?**

- Only one parameter can come from the body.
- The body must be JSON with `Content-Type: application/json`. Anything else is rejected with **415**.
- Malformed JSON is rejected with **400**.
- A body is not inferred for methods that normally have none. I mapped `app.MapGet("/search", (Filter f) => f)`: the first request returned 500, and the log said `Body was inferred but the method does not allow inferred body parameters.` For a GET, put `[AsParameters]` on the complex type to bind its members from the query string, as the example does.

**Q 8.5. What is `[AsParameters]`?**

It tells the framework to treat each constructor parameter or property of a type as if it were a separate handler parameter. One type can then collect route values, query values, headers and services. It keeps handler signatures short when a search endpoint has many filters.

**Q 8.6. How can my own type bind from a route or query value?**

Give it a static `TryParse(string, IFormatProvider, out T)` method (or the two-parameter form). The framework calls it, and a `false` result becomes a 400. `Point` in the example binds from `3,4`. For binding that needs the whole request, a static `BindAsync(HttpContext)` method is the alternative.

### Example: `08-binding.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.Mvc;   // [FromHeader], [FromQuery], [FromBody], [FromRoute], [FromServices]

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<PriceCalculator>();
var app = builder.Build();

// One handler, four binding sources. No attributes needed except for the header.
app.MapGet("/items/{id:int}", (
    int id,                                           // route: the name matches {id}
    string? q,                                        // query string; nullable = optional
    [FromHeader(Name = "X-Tenant")] string? tenant,   // header: always explicit
    PriceCalculator calculator,                       // DI: the type is a registered service
    int page = 1) =>                                  // query string with a default value
    new { id, q, page, tenant, price = calculator.PriceFor(id) });

// Repeated query keys bind to an array: /tags?tag=a&tag=b
app.MapGet("/tags", (string[] tag) => tag);

// A complex type on POST/PUT/PATCH is read from the JSON request body.
app.MapPost("/orders", (CreateOrder order) =>
    new { received = order, total = order.Quantity * order.UnitPrice });

// [AsParameters] gathers several sources into one type.
app.MapGet("/search", ([AsParameters] SearchRequest request) => request);

// A custom type binds from a route or query value if it has a static TryParse method.
app.MapGet("/map/{point}", (Point point) => new { point.X, point.Y });

// These types are supplied by the framework itself.
app.MapGet("/context", (HttpContext context, CancellationToken cancellationToken) => new
{
    context.Request.Method,
    Path = context.Request.Path.Value,
    cancellationToken.CanBeCanceled,
});

app.Run();

record CreateOrder(string Sku, int Quantity, decimal UnitPrice);

record struct SearchRequest(
    string Term,                                      // required query value
    [FromHeader(Name = "X-Tenant")] string? Tenant,
    int Page = 1,
    int PageSize = 20);

record Point(int X, int Y)
{
    public static bool TryParse(string? value, IFormatProvider? provider, out Point? point)
    {
        string[] parts = (value ?? "").Split(',');
        if (parts.Length == 2 && int.TryParse(parts[0], out int x) && int.TryParse(parts[1], out int y))
        {
            point = new Point(x, y);
            return true;
        }
        point = null;
        return false;
    }
}

class PriceCalculator
{
    public decimal PriceFor(int id) => id * 1.5m;
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 08-binding.cs

$ curl -s "http://localhost:5000/items/7?q=shoes&page=2" -H "X-Tenant: acme"
{"id":7,"q":"shoes","page":2,"tenant":"acme","price":10.5}

$ curl -s http://localhost:5000/items/7
{"id":7,"q":null,"page":1,"tenant":null,"price":10.5}

$ curl -s "http://localhost:5000/tags?tag=a&tag=b"
["a","b"]

$ curl -s -X POST http://localhost:5000/orders -H "Content-Type: application/json" -d '{"sku":"A1","quantity":3,"unitPrice":2.5}'
{"received":{"sku":"A1","quantity":3,"unitPrice":2.5},"total":7.5}

$ curl -s "http://localhost:5000/search?term=boots&pageSize=5" -H "X-Tenant: acme"
{"term":"boots","tenant":"acme","page":1,"pageSize":5}

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/search
400

$ curl -s http://localhost:5000/map/3,4
{"x":3,"y":4}

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/map/oops
400

$ curl -s http://localhost:5000/context
{"method":"GET","path":"/context","canBeCanceled":true}

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/orders -H "Content-Type: application/json" -d '{not json'
400

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/orders -H "Content-Type: text/plain" -d 'hello'
415
```

**What to notice**

- First request: `id` from the route, `q` and `page` from the query string, `tenant` from a header, `price` computed by an injected service.
- Second request: the optional values became `null`, and `page` took its default of 1.
- `/search` without `term` is 400. `/map/oops` is 400 because `TryParse` returned false.
- Invalid JSON gives 400; a non-JSON content type gives 415. The handler did not run in either case.

---

## 9. Responses: results and status codes

**Builds on:** 1, 8.

**Q 9.1. What can a handler return?**

| Return value | Response |
|---|---|
| `string` | 200, `text/plain` |
| Any other object | 200, serialized as JSON |
| An `IResult` (from `Results.*` or `TypedResults.*`) | Whatever the result says: status, headers, body |
| `Task<T>` / `ValueTask<T>` of any of these | The same, awaited |

Returning an object is enough when the answer is always 200. As soon as an endpoint can answer in more than one way (found or not found), return results.

**Q 9.2. What is the difference between `Results` and `TypedResults`?**

Both create the same responses. The difference is the declared type.

- `Results.Ok(book)` returns the interface `IResult`. The framework cannot tell from the signature what the endpoint may return.
- `TypedResults.Ok(book)` returns the concrete type `Ok<Book>`. When an endpoint has several outcomes you declare them all: `Results<Ok<Book>, NotFound>`. The compiler then rejects a result type you did not declare.

Two consequences. First, typed results describe themselves to API documentation tools. The `/metadata` endpoint in the example proves it: the typed endpoint advertises `200 Book` and `404`, the untyped one advertises nothing. Second, unit tests can assert on the concrete type without running a server.

Prefer `TypedResults`.

**Q 9.3. Which status code for which operation?**

| Operation | Success | Common failures |
|---|---|---|
| GET one | 200 with the resource | 404 |
| GET list | 200 with an array (empty array, not 404, when nothing matches) | 400 for bad query values |
| POST create | 201 with the resource and a `Location` header | 400, 409 |
| PUT replace | 204, or 200 with the updated resource | 400, 404, 409, 412 |
| PATCH | 200 or 204 | 400, 404, 409 |
| DELETE | 204 | 404 |
| Start long work | 202 with a `Location` to poll | 400 |

**Q 9.4. How do I build the `Location` header for a 201?**

Name the "get one" endpoint and use `TypedResults.CreatedAtRoute(value, routeName, routeValues)`. The framework generates the URL from the route, so it cannot go out of date. In the output the header is an absolute URL built from the request's scheme and host: `Location: http://localhost:5000/typed/books/2`. Behind a reverse proxy this is one reason forwarded headers matter (section 26).

### Example: `09-results.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Collections.Concurrent;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Http.Metadata;

var app = WebApplication.CreateBuilder(args).Build();

var books = new ConcurrentDictionary<int, Book>();
books[1] = new Book(1, "Dune");
int nextId = 1;

// 1. Return a value: an object becomes 200 + JSON, a string becomes 200 + text/plain.
app.MapGet("/plain", () => new Book(99, "Returned directly"));

// 2. Results.* returns IResult, so one handler can produce different status codes.
app.MapGet("/books/{id:int}", (int id) =>
    books.TryGetValue(id, out Book? book) ? Results.Ok(book) : Results.NotFound());

// 3. TypedResults.* does the same, but the possible responses are part of the return TYPE.
app.MapGet("/typed/books/{id:int}", Results<Ok<Book>, NotFound> (int id) =>
    books.TryGetValue(id, out Book? book) ? TypedResults.Ok(book) : TypedResults.NotFound())
    .WithName("GetBook");

// 4. Creating a resource: 201 Created plus a Location header pointing at the new resource.
app.MapPost("/books", (NewBook input) =>
{
    var book = new Book(Interlocked.Increment(ref nextId), input.Title);
    books[book.Id] = book;
    return TypedResults.CreatedAtRoute(book, "GetBook", new { id = book.Id });
});

// 5. Success with nothing to return: 204 No Content.
app.MapDelete("/books/{id:int}", (int id) =>
    books.TryRemove(id, out _) ? Results.NoContent() : Results.NotFound());

// 6. Other common outcomes.
app.MapGet("/conflict", () => Results.Conflict(new { error = "Title already exists" }));
app.MapGet("/accepted", () => Results.Accepted("/jobs/17"));
app.MapGet("/custom", () => Results.StatusCode(StatusCodes.Status418ImATeapot));

// Shows which endpoints advertise their response types to tools such as OpenAPI generators.
app.MapGet("/metadata", (EndpointDataSource source) =>
    source.Endpoints.OfType<RouteEndpoint>().ToDictionary(
        e => $"{e.Metadata.GetMetadata<HttpMethodMetadata>()!.HttpMethods[0]} {e.RoutePattern.RawText}",
        e => e.Metadata.OfType<IProducesResponseTypeMetadata>()
              .Select(m => $"{m.StatusCode} {m.Type?.Name}".Trim())));

app.Run();

record Book(int Id, string Title);
record NewBook(string Title);
```

### Run it and verify

```text
# terminal 1:  dotnet run 09-results.cs

$ curl -si http://localhost:5000/plain
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"id":99,"title":"Returned directly"}

$ curl -si http://localhost:5000/books/1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"id":1,"title":"Dune"}

$ curl -si http://localhost:5000/books/404
HTTP/1.1 404 Not Found
Content-Length: 0


$ curl -si http://localhost:5000/typed/books/1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"id":1,"title":"Dune"}

$ curl -si -X POST http://localhost:5000/books -H "Content-Type: application/json" -d '{"title":"Emma"}'
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Location: http://localhost:5000/typed/books/2

{"id":2,"title":"Emma"}

$ curl -si -X DELETE http://localhost:5000/books/2
HTTP/1.1 204 No Content


$ curl -si http://localhost:5000/conflict
HTTP/1.1 409 Conflict
Content-Type: application/json; charset=utf-8

{"error":"Title already exists"}

$ curl -si http://localhost:5000/accepted
HTTP/1.1 202 Accepted
Content-Length: 0
Location: /jobs/17


$ curl -si http://localhost:5000/custom
HTTP/1.1 418 I'm a teapot
Content-Length: 0


$ curl -s http://localhost:5000/metadata
{"GET /plain":["200 Book"],"GET /books/{id:int}":[],"GET /typed/books/{id:int}":["200 Book","404 Void"],"POST /books":["201 Book"],"DELETE /books/{id:int}":[],"GET /conflict":[],"GET /accepted":[],"GET /custom":[],"GET /metadata":["200 Dictionary`2"]}
```

**What to notice**

- 404, 204 and 418 have no body. 409 has one because the code passed an object to `Conflict`.
- `Accepted("/jobs/17")` wrote the string as given, so that `Location` is relative. `CreatedAtRoute` generated an absolute one.
- In the `/metadata` output, endpoints that return `Results.*` have empty lists. `404 Void` means a 404 with no body type. `/plain` advertises `200 Book` because its return type is the object itself.

---

## 10. JSON serialization and DTOs

**Builds on:** 8, 9.

**Q 10.1. Which serializer is used, and with what defaults?**

`System.Text.Json`, with its "web" defaults. The `/defaults` endpoint in the example reads them from the running app:

- Property names are written in **camelCase**.
- Property names are matched **case-insensitively** when reading.
- Numbers are also accepted **as strings** when reading (`"id":"7"`).

The third request below sends `ID`, `SKU` and `"7"`, and it binds. It also sends a property the type does not have (`unknown`), which is ignored.

**Q 10.2. How do I change the JSON settings?**

There are two separate option objects:

- Minimal APIs: `builder.Services.ConfigureHttpJsonOptions(...)`
- Controllers: `builder.Services.AddControllers().AddJsonOptions(...)`

Configuring one does not affect the other. I tested an app that added `JsonStringEnumConverter` only through `ConfigureHttpJsonOptions`: the minimal endpoint returned `{"color":"Green"}` and the controller returned `{"color":1}`. If you use both styles, configure both.

**Q 10.3. Which customizations come up most?**

- `JsonStringEnumConverter`: enums as names instead of numbers.
- `DefaultIgnoreCondition = WhenWritingNull`: leave out null properties.
- `[JsonPropertyName("order_total")]`: a specific name for one property.
- `[JsonIgnore]`: never write this property.
- `DateOnly`, `TimeOnly`, `DateTime`, `DateTimeOffset` and `Guid` are handled without configuration. Dates are written in ISO 8601 form (`"2026-01-15"`).

**Q 10.4. What happens when the client leaves out a property?**

By default, nothing. The property gets its default value: `null`, `0`, the first enum member, `0001-01-01`. The fourth request sends only `{"id":8}` to a record whose `Sku` is a non-nullable `string`, and it is accepted: `sku` is null (and so omitted from the echo), `status` is `Pending`, `placedOn` is `0001-01-01`. The C# nullability annotation is not enforced by the serializer by default.

Ways to close the gap, all of which I checked:

- Validation attributes such as `[Required]` (section 13). This gives the client the best error message.
- The C# `required` modifier on a property: a missing value was rejected with 400.
- `JsonSerializerOptions.RespectRequiredConstructorParameters = true` (.NET 9+): deserializing `{"id":8}` into `record Rec(int Id, string Sku)` failed with `JSON deserialization for type 'Rec' was missing required properties including: 'sku'.` Its companion `RespectNullableAnnotations` rejects an explicit `null` for a non-nullable property. Both are off by default.

**Q 10.5. Why use separate DTOs (request and response types) and not my entity classes?**

- **Over-posting.** If the bound type has `IsAdmin` or `CreatedBy`, a client can set them by adding them to the JSON.
- **Leaking.** Returning an entity exposes every field, including ones added later.
- **Coupling.** The JSON shape is a public contract. The storage shape changes for internal reasons. Separate types let each change without breaking the other.

Section 11 applies this.

### Example: `10-json.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Text.Json.Serialization;
using Microsoft.AspNetCore.Http.Json;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// JSON settings for minimal APIs. (Controllers have their own: AddControllers().AddJsonOptions.)
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());                // "Shipped", not 1
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;  // leave nulls out
});

var app = builder.Build();

app.MapGet("/order", () =>
    new Order(1, "A-100", OrderStatus.Shipped, Note: null, new DateOnly(2026, 1, 15), 12.50m, "internal-7"));

// Echo: returns whatever the JSON binder produced from the request body.
app.MapPost("/order", (Order order) => order);

// The defaults ASP.NET Core applies before any customisation.
app.MapGet("/defaults", (IOptions<JsonOptions> options) => new
{
    namingPolicy = options.Value.SerializerOptions.PropertyNamingPolicy == System.Text.Json.JsonNamingPolicy.CamelCase
        ? "camelCase" : "other",
    caseInsensitive = options.Value.SerializerOptions.PropertyNameCaseInsensitive,
    numberHandling = options.Value.SerializerOptions.NumberHandling.ToString(),
});

app.Run();

enum OrderStatus { Pending, Shipped, Delivered }

record Order(
    int Id,
    string Sku,
    OrderStatus Status,
    string? Note,
    DateOnly PlacedOn,
    [property: JsonPropertyName("order_total")] decimal Total,
    [property: JsonIgnore] string? InternalCode = null);
```

### Run it and verify

```text
# terminal 1:  dotnet run 10-json.cs

$ curl -s http://localhost:5000/order
{"id":1,"sku":"A-100","status":"Shipped","placedOn":"2026-01-15","order_total":12.50}

$ curl -s http://localhost:5000/defaults
{"namingPolicy":"camelCase","caseInsensitive":true,"numberHandling":"AllowReadingFromString"}

$ curl -s -X POST http://localhost:5000/order -H "Content-Type: application/json" -d '{"ID":"7","SKU":"B-2","status":"Delivered","placedOn":"2026-02-01","order_total":5,"unknown":true}'
{"id":7,"sku":"B-2","status":"Delivered","placedOn":"2026-02-01","order_total":5}

$ curl -s -X POST http://localhost:5000/order -H "Content-Type: application/json" -d '{"id":8}'
{"id":8,"status":"Pending","placedOn":"0001-01-01","order_total":0}

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/order -H "Content-Type: application/json" -d '{"id":9,"status":"Lost"}'
400
```

**What to notice**

- In the GET response: `status` is the string `"Shipped"`, `note` is absent (it was null), `order_total` uses the custom name, and `internalCode` is absent because of `[JsonIgnore]`.
- An enum name that does not exist (`"Lost"`) is a deserialization error, so the framework answers 400.

---

## 11. A complete CRUD resource with route groups

**Builds on:** 3, 8, 9, 10.

This section adds one new feature, route groups, and otherwise assembles what came before into the shape most REST endpoints have.

**Q 11.1. What is a route group?**

`app.MapGroup("/api/todos")` returns a builder. Endpoints mapped on it share the prefix, and anything applied to the group applies to all of them: `RequireAuthorization()`, `AddEndpointFilter(...)`, `RequireRateLimiting(...)`, `WithTags(...)`. Groups can be nested. They replace repeating the same prefix and the same calls on every endpoint.

**Q 11.2. What should each operation return?**

The example follows the table in section 9: the list is always 200 (an empty array at first), get-one is 200 or 404, create is 201 with a `Location`, replace and delete are 204 or 404.

**Q 11.3. Why is there a repository interface?**

Handlers ask for `ITodoRepository` and do not know how data is stored. Replacing the in-memory class with a database-backed one (section 28) changes one registration line and no handler. Tests can supply a fake.

**Q 11.4. Why is the repository a singleton here, and is that normal?**

It holds the data in memory, so it has to live as long as the application. That also means many requests use it at the same time, so it uses `ConcurrentDictionary` and `Interlocked`. A repository built on a database session is registered as **scoped**, because the session is scoped.

**Q 11.5. Should PUT create the resource if it does not exist?**

HTTP allows it, and then PUT would return 201. Most APIs that generate ids on the server do not allow it, and return 404 as the example does. Pick one behaviour and document it.

### Example: `11-crud.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Collections.Concurrent;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ITodoRepository, InMemoryTodoRepository>();
var app = builder.Build();

// A route group: one shared prefix (and shared settings) for a set of endpoints.
RouteGroupBuilder todos = app.MapGroup("/api/todos");

todos.MapGet("/", (ITodoRepository repo) =>
    TypedResults.Ok(repo.All().Select(TodoResponse.From)));

todos.MapGet("/{id:int}", Results<Ok<TodoResponse>, NotFound> (int id, ITodoRepository repo) =>
    repo.Find(id) is { } todo ? TypedResults.Ok(TodoResponse.From(todo)) : TypedResults.NotFound())
    .WithName("GetTodo");

todos.MapPost("/", (TodoRequest request, ITodoRepository repo) =>
{
    Todo todo = repo.Add(request.Title, request.IsDone);
    return TypedResults.CreatedAtRoute(TodoResponse.From(todo), "GetTodo", new { id = todo.Id });
});

todos.MapPut("/{id:int}", Results<NoContent, NotFound> (int id, TodoRequest request, ITodoRepository repo) =>
    repo.Update(id, request.Title, request.IsDone) ? TypedResults.NoContent() : TypedResults.NotFound());

todos.MapDelete("/{id:int}", Results<NoContent, NotFound> (int id, ITodoRepository repo) =>
    repo.Remove(id) ? TypedResults.NoContent() : TypedResults.NotFound());

app.Run();

// -- API contracts (DTOs): the shape clients see. Deliberately separate from the entity.
record TodoRequest(string Title, bool IsDone);

record TodoResponse(int Id, string Title, bool IsDone)
{
    public static TodoResponse From(Todo todo) => new(todo.Id, todo.Title, todo.IsDone);
}

// -- Entity: the shape the application stores. It has fields clients must never set.
class Todo
{
    public int Id { get; init; }
    public required string Title { get; set; }
    public bool IsDone { get; set; }
    public DateTime CreatedUtc { get; init; }
    public string CreatedBy { get; init; } = "system";
}

// -- Data access hidden behind an interface, supplied by dependency injection.
interface ITodoRepository
{
    IEnumerable<Todo> All();
    Todo? Find(int id);
    Todo Add(string title, bool isDone);
    bool Update(int id, string title, bool isDone);
    bool Remove(int id);
}

class InMemoryTodoRepository : ITodoRepository
{
    readonly ConcurrentDictionary<int, Todo> _items = new();
    int _nextId;

    public IEnumerable<Todo> All() => _items.Values.OrderBy(t => t.Id);

    public Todo? Find(int id) => _items.GetValueOrDefault(id);

    public Todo Add(string title, bool isDone)
    {
        var todo = new Todo
        {
            Id = Interlocked.Increment(ref _nextId),
            Title = title,
            IsDone = isDone,
            CreatedUtc = DateTime.UtcNow,
        };
        _items[todo.Id] = todo;
        return todo;
    }

    public bool Update(int id, string title, bool isDone)
    {
        if (!_items.TryGetValue(id, out Todo? todo)) return false;
        todo.Title = title;
        todo.IsDone = isDone;
        return true;
    }

    public bool Remove(int id) => _items.TryRemove(id, out _);
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 11-crud.cs

$ curl -s http://localhost:5000/api/todos
[]

$ curl -si -X POST http://localhost:5000/api/todos -H "Content-Type: application/json" -d '{"title":"Write docs","isDone":false}'
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Location: http://localhost:5000/api/todos/1

{"id":1,"title":"Write docs","isDone":false}

$ curl -s http://localhost:5000/api/todos/1
{"id":1,"title":"Write docs","isDone":false}

$ curl -si -X PUT http://localhost:5000/api/todos/1 -H "Content-Type: application/json" -d '{"title":"Write docs","isDone":true}'
HTTP/1.1 204 No Content


$ curl -s http://localhost:5000/api/todos
[{"id":1,"title":"Write docs","isDone":true}]

$ curl -si -X DELETE http://localhost:5000/api/todos/1
HTTP/1.1 204 No Content


$ curl -si http://localhost:5000/api/todos/1
HTTP/1.1 404 Not Found
Content-Length: 0


$ curl -s -o /dev/null -w "%{http_code}\n" -X PUT http://localhost:5000/api/todos/99 -H "Content-Type: application/json" -d '{"title":"x","isDone":true}'
404
```

**What to notice**

- The response never contains `createdUtc` or `createdBy`. They exist on the entity but not on `TodoResponse`.
- The client cannot set `id`: `TodoRequest` has no such property.

---

## 12. Controllers

**Builds on:** 3, 7, 9, 10.

**Q 12.1. What is a controller, and how does it differ from minimal APIs?**

A controller is a class whose public methods (**actions**) are endpoints. Routes and HTTP methods are declared with attributes. Controllers and minimal APIs run on the same routing, DI, middleware and security. They differ in organization and in a few features.

Microsoft's guidance: "For new projects, we recommend using Minimal APIs". The same page lists reasons to choose controllers: model binding extensibility (`IModelBinder`), form binding including `IFormFile`, advanced validation features (`IModelValidator`), application parts and the application model, and OData. You will also meet controllers in most existing codebases, so you need to read both.

**Q 12.2. What does `[ApiController]` do?**

It turns on API-specific behaviour for the controller:

- **Attribute routing is required.** Actions are reachable only through `[Route]`, `[HttpGet]` and similar attributes.
- **Automatic 400.** If binding or validation fails, the action does not run and the client gets a 400 with the errors.
- **Binding source inference.** Complex types come from the body; a parameter whose name is in the route template comes from the route; other simple types come from the query string; types registered in DI are injected; `IFormFile` comes from the form.
- **Problem details for error status codes.** `return NotFound();` produces a JSON body, not an empty response.

**Q 12.3. Why derive from `ControllerBase` and not `Controller`?**

`Controller` adds support for rendering HTML views. An API does not need it. `ControllerBase` provides everything an API uses: `Ok()`, `NotFound()`, `CreatedAtAction()`, `User`, `Request`, `ModelState`.

**Q 12.4. What should an action return?**

- `T` when the action always succeeds with a body.
- `ActionResult<T>` when it returns either a `T` or another result such as `NotFound()`. The type documents the success body and lets you `return product;` directly.
- `IActionResult` when there is no body type to declare (for example a delete).

**Q 12.5. What is the lifetime of a controller?**

A new controller instance is created for every request. Constructor-injected scoped services are therefore safe.

**Q 12.6. What does `[controller]` in the route mean?**

It is replaced by the class name without the `Controller` suffix. `ProductsController` gives `api/Products`. Matching is case-insensitive, so `/api/products` works. Generated URLs use the class-name casing, as the `Location` header below shows (`/api/Products/3`). Set `RouteOptions.LowercaseUrls = true` to generate lowercase URLs; it is `false` by default.

### Example: `12-controllers.cs`

Controllers must be `public` classes to be discovered.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Collections.Concurrent;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();              // MVC services for API controllers (no Razor views)
builder.Services.AddSingleton<ProductStore>();
var app = builder.Build();

app.MapControllers();                           // turn attribute-routed actions into endpoints
app.Run();

[ApiController]                                 // opt in to the web API conventions
[Route("api/[controller]")]                     // [controller] = class name minus "Controller" -> api/products
public class ProductsController(ProductStore store) : ControllerBase
{
    [HttpGet]                                   // GET api/products?category=tools
    public IEnumerable<Product> GetAll([FromQuery] string? category) =>
        store.All().Where(p => category is null || p.Category == category);

    [HttpGet("{id:int}")]                       // GET api/products/1
    public ActionResult<Product> GetById(int id)
    {
        Product? product = store.Find(id);
        return product is null ? NotFound() : product;   // ActionResult<T>: a result OR a T
    }

    [HttpPost]                                  // POST api/products  (body bound from JSON)
    [ProducesResponseType<Product>(StatusCodes.Status201Created)]
    public ActionResult<Product> Create(CreateProduct input)
    {
        Product product = store.Add(input.Name, input.Category);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpDelete("{id:int}")]                    // DELETE api/products/1
    public IActionResult Delete(int id) => store.Remove(id) ? NoContent() : NotFound();
}

public record Product(int Id, string Name, string Category);
public record CreateProduct(string Name, string Category);

public class ProductStore
{
    readonly ConcurrentDictionary<int, Product> _items = new();
    int _nextId;

    public ProductStore()
    {
        Add("Hammer", "tools");
        Add("Notebook", "stationery");
    }

    public IEnumerable<Product> All() => _items.Values.OrderBy(p => p.Id);
    public Product? Find(int id) => _items.GetValueOrDefault(id);
    public bool Remove(int id) => _items.TryRemove(id, out _);

    public Product Add(string name, string category)
    {
        var product = new Product(Interlocked.Increment(ref _nextId), name, category);
        _items[product.Id] = product;
        return product;
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 12-controllers.cs

$ curl -s http://localhost:5000/api/products
[{"id":1,"name":"Hammer","category":"tools"},{"id":2,"name":"Notebook","category":"stationery"}]

$ curl -s "http://localhost:5000/api/products?category=tools"
[{"id":1,"name":"Hammer","category":"tools"}]

$ curl -s http://localhost:5000/api/products/1
{"id":1,"name":"Hammer","category":"tools"}

$ curl -si http://localhost:5000/api/products/99
HTTP/1.1 404 Not Found
Content-Type: application/problem+json; charset=utf-8

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Not Found","status":404,"traceId":"00-b826eda09a0ff83df91adab7dcda2d2a-8cd8fab5590b183c-00"}

$ curl -si -X POST http://localhost:5000/api/products -H "Content-Type: application/json" -d '{"name":"Pen","category":"stationery"}'
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Location: http://localhost:5000/api/Products/3

{"id":3,"name":"Pen","category":"stationery"}

$ curl -si -X DELETE http://localhost:5000/api/products/3
HTTP/1.1 204 No Content


$ curl -si -X POST http://localhost:5000/api/products -H "Content-Type: application/json" -d '{broken'
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json; charset=utf-8

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"$":["'b' is an invalid start of a property name. Expected a '\"'. Path: $ | LineNumber: 0 | BytePositionInLine: 1."],"input":["The input field is required."]},"traceId":"00-52e83ea98d841a12b4fb51c4f90aa92b-572a3860d4e6dbfe-00"}

$ curl -s -X POST http://localhost:5000/api/products -H "Content-Type: application/json" -d '{}'
{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"Name":["The Name field is required."],"Category":["The Category field is required."]},"traceId":"00-d5426bb956e43db4b2d5224ef674c8f0-29a4eda069ae71dd-00"}
```

**What to notice**

- The 404 has an `application/problem+json` body, although the code only said `NotFound()`. Compare with the empty 404 from the minimal API in section 9. Section 14 explains the format and how to get it everywhere.
- Malformed JSON produced a 400 with details and the action never ran.
- The last request sent `{}`. `Name` and `Category` are non-nullable `string`s, and MVC treats non-nullable reference types as required, so the response lists both as missing. No attribute was needed. (Minimal APIs do not do this; see Q 10.4.)

---

# Part 4: Making it robust

## 13. Validation

**Builds on:** 8, 12.

**Q 13.1. What is validation, and when does it run?**

Binding answers "can this request be turned into the parameter types?". Validation answers "are the values acceptable?": lengths, ranges, formats, rules between fields. It runs after binding and before your handler or action. If it fails, your code does not run and the client gets a **400** listing every problem.

**Q 13.2. How do I declare the rules?**

With attributes from `System.ComponentModel.DataAnnotations` on properties or parameters: `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]`, `[Url]`, `[Compare]` and others. For a rule that involves more than one property, implement `IValidatableObject` on the type and return errors from `Validate`.

**Q 13.3. Who runs the validation?**

- **Controllers:** the framework always validates bound models, and `[ApiController]` turns a failure into an automatic 400.
- **Minimal APIs on .NET 10:** call `builder.Services.AddValidation()`. A source generator finds the types used by your endpoints and an endpoint filter validates them. It only finds types in the assembly where `AddValidation` is called. `.DisableValidation()` opts an endpoint out.
- **Minimal APIs before .NET 10:** there is no built-in validation. You validate in the handler, write an endpoint filter (section 15), or use a library.

**Q 13.4. What does the error response look like?**

A problem-details document with an `errors` object: property name to list of messages. Both styles produce it, as the first and last requests below show.

One detail for minimal APIs: I first ran the example without `builder.Services.AddProblemDetails()`. The 400 then came back as plain `application/json` containing only `title` and `errors`. After adding `AddProblemDetails()` it became `application/problem+json` with `type`, `status` and `traceId` as well. Register it (section 14 covers what else it does).

**Q 13.5. When does `IValidatableObject.Validate` run?**

Only after the attribute rules pass. The third request below has a too-short name and reversed dates: only the name error is reported. The second request has valid attributes and reversed dates: the cross-field error appears.

**Q 13.6. Is every failed rule a 400?**

No. 400 is for input that is malformed or breaks a rule you can check from the request alone. A rule that depends on current server state ("this email is already registered", "not enough stock") is not a validation error in this sense. Return 409 Conflict, or 422, from the handler with a problem-details body (section 14).

Validate on the server even when the client also validates. The client is not under your control.

### Example: `13-validation.cs`

The same `Customer` type is posted to a minimal endpoint and to a controller.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddValidation();      // .NET 10: validate minimal API parameters with DataAnnotations
builder.Services.AddControllers();     // controllers validate automatically through [ApiController]
builder.Services.AddProblemDetails();  // write validation failures as RFC 9457 problem details
var app = builder.Build();

// Minimal API: the body is validated before the handler runs.
app.MapPost("/minimal/customers", (Customer customer) => TypedResults.Ok(customer));

// Attributes also work directly on simple parameters.
app.MapGet("/minimal/customers", ([Range(1, 100)] int pageSize) => $"pageSize={pageSize}");

// Opt a single endpoint out.
app.MapPost("/minimal/unvalidated", (Customer customer) => TypedResults.Ok(customer))
   .DisableValidation();

app.MapControllers();
app.Run();

[ApiController]
[Route("mvc/customers")]
public class CustomersController : ControllerBase
{
    [HttpPost]
    public ActionResult<Customer> Create(Customer customer) => customer;   // only reached when valid
}

public class Customer : IValidatableObject
{
    [Required, StringLength(50, MinimumLength = 2)]
    public string? Name { get; set; }

    [Required, EmailAddress]
    public string? Email { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }

    public DateOnly? ValidFrom { get; set; }
    public DateOnly? ValidTo { get; set; }

    // Cross-property rule: something no single attribute can express.
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (ValidFrom > ValidTo)
        {
            yield return new ValidationResult("ValidTo must not be before ValidFrom.", [nameof(ValidTo)]);
        }
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 13-validation.cs

$ curl -si -X POST http://localhost:5000/minimal/customers -H "Content-Type: application/json" -d '{"name":"A","email":"nope","age":7}'
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"Name":["The field Name must be a string with a minimum length of 2 and a maximum length of 50."],"Email":["The Email field is not a valid e-mail address."],"Age":["The field Age must be between 18 and 120."]},"traceId":"00-972da3ac1dab561107eeadfba242fb54-f1981f08c953d54a-00"}

$ curl -s -X POST http://localhost:5000/minimal/customers -H "Content-Type: application/json" -d '{"name":"Asha","email":"asha@example.com","age":30,"validFrom":"2026-05-01","validTo":"2026-01-01"}'
{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"ValidTo":["ValidTo must not be before ValidFrom."]},"traceId":"00-5e6267652068a88fab4ab9328bf22699-13b5d54831eb2112-00"}

$ curl -s -X POST http://localhost:5000/minimal/customers -H "Content-Type: application/json" -d '{"name":"A","email":"asha@example.com","age":30,"validFrom":"2026-05-01","validTo":"2026-01-01"}'
{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"Name":["The field Name must be a string with a minimum length of 2 and a maximum length of 50."]},"traceId":"00-1e252a38de76326ddacfa46dbc9d8a6c-e1d1711addf0a13d-00"}

$ curl -s -X POST http://localhost:5000/minimal/customers -H "Content-Type: application/json" -d '{"name":"Asha","email":"asha@example.com","age":30}'
{"name":"Asha","email":"asha@example.com","age":30,"validFrom":null,"validTo":null}

$ curl -s "http://localhost:5000/minimal/customers?pageSize=500"
{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"pageSize":["The field pageSize must be between 1 and 100."]},"traceId":"00-30f6e138e0aef0a082da87c873e9cd31-4c2f767989a6ffd2-00"}

$ curl -s -X POST http://localhost:5000/minimal/unvalidated -H "Content-Type: application/json" -d '{"name":"A","email":"nope","age":7}'
{"name":"A","email":"nope","age":7,"validFrom":null,"validTo":null}

$ curl -si -X POST http://localhost:5000/mvc/customers -H "Content-Type: application/json" -d '{"name":"A","email":"nope","age":7}'
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json; charset=utf-8

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.1","title":"One or more validation errors occurred.","status":400,"errors":{"Age":["The field Age must be between 18 and 120."],"Name":["The field Name must be a string with a minimum length of 2 and a maximum length of 50."],"Email":["The Email field is not a valid e-mail address."]},"traceId":"00-287ffe099c1a327d9075e1e153f9e999-1b98e0ec0d5295b3-00"}
```

**What to notice**

- All three attribute failures are reported at once in the first response.
- `[Range]` on the `pageSize` query parameter was enforced too.
- `/minimal/unvalidated` accepted the same invalid body because validation is disabled for it.
- The controller's response has the same shape as the minimal API's.

---

## 14. Error handling and problem details

**Builds on:** 6, 9.

**Q 14.1. What happens to an unhandled exception if I do nothing?**

In Production, the client gets a 500 with an empty body and the exception is logged. I checked this with an endpoint that throws: `HTTP/1.1 500 Internal Server Error`, `Content-Length: 0`. In Development, the developer exception page shows the exception, stack trace and request details. That page must never be reachable in production.

An empty 500 is safe, but unhelpful and inconsistent with your other error responses.

**Q 14.2. What is "problem details"?**

A standard JSON format for API errors, defined by RFC 9457 (which replaced RFC 7807). Media type `application/problem+json`. The fields:

| Field | Meaning |
|---|---|
| `type` | A URI identifying the kind of problem |
| `title` | Short, human-readable summary of the kind of problem |
| `status` | The HTTP status code |
| `detail` | Explanation of this particular occurrence |
| `instance` | A URI for this occurrence |
| anything else | Extension members, such as `traceId` or `errors` |

Using it for every error means clients need one error parser.

**Q 14.3. Which three calls give me consistent errors?**

- `builder.Services.AddProblemDetails()` registers the service that writes problem details. Its `CustomizeProblemDetails` callback is the one place to add fields to all of them.
- `app.UseExceptionHandler()` catches exceptions from everything after it and writes a 500 problem-details response.
- `app.UseStatusCodePages()` adds a problem-details body to error responses that have none, such as the 404 for an unknown route.

**Q 14.4. How do I map specific exceptions to specific responses?**

Implement `IExceptionHandler` (.NET 8+) and register it with `AddExceptionHandler<T>()`. Handlers are singletons and run in registration order. Return `true` if you wrote the response, `false` to pass the exception on. If none handles it, the default 500 is written.

On .NET 10 the middleware no longer logs an exception that a handler dealt with. The log excerpt below confirms it: `/boom` is logged at error level, the handled `OrderNotFoundException` from `/orders/7` is not. If you want those logged, log in the handler.

**Q 14.5. Should I throw exceptions for expected outcomes like "not found"?**

Prefer returning a result. `TypedResults.NotFound()` or `TypedResults.Problem(...)` is explicit, shows up in the endpoint's declared results, and costs less than throwing. Keep exceptions for the unexpected. The exception-to-response mapping in the example is useful mainly when the failure is detected deep in code that has no access to the HTTP layer.

**Q 14.6. What must an error response never contain?**

Exception messages, stack traces, SQL, file paths, connection strings. The `/boom` endpoint throws an exception whose message contains a password. The response body below does not contain it. The log does, so access to logs must be controlled as well.

### Example: `14-errors.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// One place that shapes every ProblemDetails response the framework writes.
builder.Services.AddProblemDetails(options =>
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Instance = context.HttpContext.Request.Path;
        context.ProblemDetails.Extensions["service"] = "orders-api";
    });

// Exception handlers run in registration order until one returns true.
builder.Services.AddExceptionHandler<OrderNotFoundHandler>();

var app = builder.Build();

app.UseExceptionHandler();    // catch unhandled exceptions -> handlers above -> ProblemDetails
app.UseStatusCodePages();     // give body-less 4xx/5xx responses (e.g. a routing 404) a ProblemDetails body

app.MapGet("/ok", () => "fine");

// An unexpected failure. The message contains something a client must never see.
app.MapGet("/boom", string () =>
    throw new InvalidOperationException("Connection failed: Server=db01;Password=hunter2"));

// An expected, domain-level failure mapped to 404 by OrderNotFoundHandler.
app.MapGet("/orders/{id:int}", string (int id) => throw new OrderNotFoundException(id));

// No exception at all: return a problem deliberately.
app.MapGet("/stock", () => TypedResults.Problem(
    title: "Insufficient stock",
    detail: "Only 2 units of SKU A-100 are available.",
    statusCode: StatusCodes.Status409Conflict));

app.Run();

class OrderNotFoundException(int id) : Exception($"Order {id} does not exist.")
{
    public int Id { get; } = id;
}

class OrderNotFoundHandler(IProblemDetailsService problemDetails) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        if (exception is not OrderNotFoundException notFound)
        {
            return false;   // not mine: let the next handler (or the default 500) deal with it
        }

        httpContext.Response.StatusCode = StatusCodes.Status404NotFound;
        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Status = StatusCodes.Status404NotFound,
                Title = "Order not found",
                Detail = notFound.Message,
            },
        });
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 14-errors.cs

$ curl -si http://localhost:5000/boom
HTTP/1.1 500 Internal Server Error
Content-Type: application/problem+json
Cache-Control: no-cache,no-store
Expires: -1
Pragma: no-cache

{"type":"https://tools.ietf.org/html/rfc9110#section-15.6.1","title":"An error occurred while processing your request.","status":500,"instance":"/boom","traceId":"00-0b4358c84c0c1531cd5469b02996f5d8-9b8767279b927a45-00","service":"orders-api"}

$ curl -si http://localhost:5000/orders/7
HTTP/1.1 404 Not Found
Content-Type: application/problem+json
Cache-Control: no-cache,no-store
Expires: -1
Pragma: no-cache

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Order not found","status":404,"detail":"Order 7 does not exist.","instance":"/orders/7","traceId":"00-0bdbb27110a273044816e53871d724eb-2906193d3a65b96d-00","service":"orders-api"}

$ curl -si http://localhost:5000/stock
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.10","title":"Insufficient stock","status":409,"detail":"Only 2 units of SKU A-100 are available.","instance":"/stock","traceId":"00-f7783141bec3659155ee9a7ba6d56dc5-4805d12275ab55c1-00","service":"orders-api"}

$ curl -si http://localhost:5000/no-such-route
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Not Found","status":404,"instance":"/no-such-route","traceId":"00-76067a1fff6a2ea886f46bf8c5d7a83f-cd73a243dd67dc05-00","service":"orders-api"}

--- what the app logged at error level ---
fail: Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware[1]
      An unhandled exception has occurred while executing the request.
      System.InvalidOperationException: Connection failed: Server=db01;Password=hunter2
         at Program.<>c.<<Main>$>b__0_2()
```

**What to notice**

- All four responses, from four different causes, have the same shape and all carry `instance` and `service` from the single `CustomizeProblemDetails` callback.
- `traceId` was added by the framework. It lets you find the matching log entries when a client reports an error.
- The 500 says only "An error occurred while processing your request."
- The exception handler's responses also got `Cache-Control: no-cache,no-store`, so a failure is not cached.

---

## 15. Filters

**Builds on:** 6, 8, 12.

**Q 15.1. How is a filter different from middleware?**

| | Middleware | Filter |
|---|---|---|
| Applies to | Every request passing through that point of the pipeline | The endpoints, groups, controllers or actions you attach it to |
| Runs | Before the endpoint is invoked, with only `HttpContext` | Inside the endpoint, after parameter binding |
| Can see | Raw request and response | The handler's typed arguments and its return value |

Use middleware for concerns that do not depend on which endpoint runs (exception handling, CORS, compression). Use a filter when the logic needs the bound arguments or the result, or applies only to some endpoints.

**Q 15.2. How do endpoint filters work in minimal APIs?**

`AddEndpointFilter` (.NET 7+) wraps the handler. The filter receives a context with the bound arguments (`context.GetArgument<T>(index)`) and a `next` delegate. It can return its own result without calling `next`, change arguments before, or change the result after. Filters run in the order added: the first added is the outermost. A filter on a group wraps filters on the endpoint.

**Q 15.3. What filters do controllers have?**

MVC has its own filter pipeline with five kinds:

1. **Authorization filters** run first and decide whether the request may continue.
2. **Resource filters** wrap everything that follows, including model binding.
3. **Action filters** run just before and after the action method. Arguments are bound; the result can be changed.
4. **Exception filters** run only if something above threw, and can turn the exception into a result.
5. **Result filters** run before and after the result is written.

A filter can be applied to one action, one controller, or globally through `AddControllers(options => options.Filters.Add(...))`.

**Q 15.4. Why can the filter in the example still set a header after `next`?**

Because an endpoint filter gets the handler's return value as an object. The response is written only after the whole filter chain returns. Compare this with middleware in section 6, where code after `next` ran after the body had been sent.

### Example: `15-filters.cs`

Each step records its name, and the outermost filter puts the sequence in an `X-Trace` header.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var app = builder.Build();

// -- Endpoint filters (minimal APIs) -----------------------------------------
// A filter on a group applies to every endpoint in the group.
RouteGroupBuilder api = app.MapGroup("/api").AddEndpointFilter<TraceFilter>();

api.MapGet("/square/{n:int}", (int n, HttpContext context) =>
    {
        Trace.Add(context, "handler");
        return Results.Ok(n * n);
    })
   .AddEndpointFilter(async (context, next) =>
   {
       // Filters see the handler's arguments AFTER model binding.
       int n = context.GetArgument<int>(0);
       if (n < 0)
       {
           Trace.Add(context.HttpContext, "inline:rejected");
           return Results.BadRequest("n must not be negative");   // short-circuit: handler never runs
       }

       Trace.Add(context.HttpContext, "inline:before");
       object? result = await next(context);                      // next filter, then the handler
       Trace.Add(context.HttpContext, "inline:after");
       return result;
   });

// -- MVC filters (controllers) -----------------------------------------------
app.MapControllers();
app.Run();

static class Trace
{
    public static void Add(HttpContext context, string step)
    {
        var steps = (List<string>)(context.Items["trace"] ??= new List<string>());
        steps.Add(step);
    }

    public static string Read(HttpContext context) =>
        string.Join(" > ", (List<string>?)context.Items["trace"] ?? []);
}

// A reusable endpoint filter class. It can take constructor dependencies from DI.
class TraceFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        Trace.Add(context.HttpContext, "group:before");
        object? result = await next(context);
        Trace.Add(context.HttpContext, "group:after");

        // The result has not been written yet, so headers can still be set here.
        context.HttpContext.Response.Headers["X-Trace"] = Trace.Read(context.HttpContext);
        return result;
    }
}

// An MVC action filter, applied as an attribute.
public class TraceActionAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context) =>
        Trace.Add(context.HttpContext, "action-filter:before");

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        Trace.Add(context.HttpContext, "action-filter:after");
        context.HttpContext.Response.Headers["X-Trace"] = Trace.Read(context.HttpContext);
    }
}

[ApiController]
[Route("mvc")]
public class PingController : ControllerBase
{
    [HttpGet("ping")]
    [TraceAction]
    public string Ping()
    {
        Trace.Add(HttpContext, "action");
        return "pong";
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 15-filters.cs

$ curl -si http://localhost:5000/api/square/4
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Trace: group:before > inline:before > handler > inline:after > group:after

16

$ curl -si http://localhost:5000/api/square/-4
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Trace: group:before > inline:rejected > group:after

"n must not be negative"

$ curl -si http://localhost:5000/mvc/ping
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
X-Trace: action-filter:before > action > action-filter:after

pong
```

**What to notice**

- Normal order: group filter, endpoint filter, handler, then back out.
- For `-4` the trace has no `handler` step. The inline filter returned 400 itself.
- The MVC action filter wraps the action in the same way.

---

## 16. Async, cancellation and timeouts

**Builds on:** 8.

**Q 16.1. Why should handlers be `async`?**

A request that waits for a database or another service does nothing useful while it waits. With `await`, the thread goes back to the thread pool during the wait and serves other requests. With a blocking call, the thread is held. Under load, blocked threads run out and requests queue even though the CPU is idle.

Async does not make one request faster. It lets the server handle more requests at once.

Rules that follow:

- Use the async version of every I/O call and `await` it, all the way up to the handler.
- Never block on a task with `.Result`, `.Wait()` or `.GetAwaiter().GetResult()` in request code.
- Do not use `async void`. Exceptions thrown from it cannot be caught by the caller.

**Q 16.2. How do I run independent operations at the same time?**

Start them, then `await Task.WhenAll(...)`. In the example three one-second operations take about 3 seconds when awaited one by one and about 1 second together. This applies to operations that are independent and safe to run concurrently. An EF Core `DbContext`, for example, does not support concurrent operations on one instance.

**Q 16.3. What is the `CancellationToken` handler parameter?**

It is `HttpContext.RequestAborted`. It is cancelled when the client disconnects. Pass it to every async call that accepts one (database queries, `HttpClient`, `Task.Delay`). Then work for a client that has gone away stops, and the resources are freed. In the example curl gives up after 2 seconds and the server logs that the 30-second operation was cancelled.

Cancellation is cooperative. Code that ignores the token keeps running.

**Q 16.4. How do I limit how long a request may take?**

With the request timeouts middleware (.NET 8+): `AddRequestTimeouts()`, `UseRequestTimeouts()`, then `.WithRequestTimeout(...)` on an endpoint or a default policy in the options. No timeout applies until you configure one. When the limit is reached, the same `RequestAborted` token is cancelled, and if the handler does not produce a response the client gets **504**.

Two documented details: the middleware does not trigger while a debugger is attached, and if you call `UseRouting` explicitly, `UseRequestTimeouts` must come after it.

**Q 16.5. May I start work and return without waiting for it?**

Not with a bare `Task.Run` or an un-awaited task. The request's scoped services are disposed when the response ends, unobserved exceptions are lost, and the work dies if the process shuts down. Hand the work to a background service (section 23).

### Example: `16-async.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Diagnostics;
using Microsoft.AspNetCore.Http.Timeouts;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRequestTimeouts();     // opt-in request timeout support (.NET 8+)
var app = builder.Build();
app.UseRequestTimeouts();

// Three independent I/O calls awaited one after another: the delays add up.
app.MapGet("/sequential", async (CancellationToken ct) =>
{
    var watch = Stopwatch.StartNew();
    await FakeIo("a", ct);
    await FakeIo("b", ct);
    await FakeIo("c", ct);
    return $"sequential: about {Math.Round(watch.Elapsed.TotalSeconds)} s";
});

// The same three calls started together and awaited as a group.
app.MapGet("/concurrent", async (CancellationToken ct) =>
{
    var watch = Stopwatch.StartNew();
    await Task.WhenAll(FakeIo("a", ct), FakeIo("b", ct), FakeIo("c", ct));
    return $"concurrent: about {Math.Round(watch.Elapsed.TotalSeconds)} s";
});

// The CancellationToken parameter is HttpContext.RequestAborted: it fires when the client goes away.
app.MapGet("/slow", async (CancellationToken ct, ILogger<Program> logger) =>
{
    try
    {
        await Task.Delay(TimeSpan.FromSeconds(30), ct);
        return "finished";
    }
    catch (OperationCanceledException)
    {
        logger.LogWarning("Client disconnected: the 30 s operation was cancelled early");
        throw;
    }
});

// A server-side limit: the same token is cancelled after 1 second and the client gets 504.
app.MapGet("/limited", async (CancellationToken ct) =>
{
    await Task.Delay(TimeSpan.FromSeconds(30), ct);
    return "finished";
}).WithRequestTimeout(TimeSpan.FromSeconds(1));

app.Run();

static async Task<string> FakeIo(string name, CancellationToken ct)
{
    await Task.Delay(TimeSpan.FromSeconds(1), ct);   // stands in for a database or HTTP call
    return name;
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 16-async.cs

$ curl -s http://localhost:5000/sequential
sequential: about 3 s

$ curl -s http://localhost:5000/concurrent
concurrent: about 1 s

$ curl -s -m 2 http://localhost:5000/slow; echo "curl exit code: $?"
curl exit code: 28

$ curl -si http://localhost:5000/limited
HTTP/1.1 504 Gateway Timeout
Content-Length: 0


--- what the app logged ---
warn: Program[0]
      Client disconnected: the 30 s operation was cancelled early
warn: Microsoft.AspNetCore.Http.Timeouts.RequestTimeoutsMiddleware[1]
      Timeout exception handled.
```

**What to notice**

- curl exit code 28 is "operation timed out". curl closed the connection, and the server's warning appears in the log.
- `/limited` returned 504 after one second without any timeout code in the handler. The log line is from the middleware.

---

# Part 5: Security and traffic control

## 17. Authentication

**Builds on:** 3, 6.

**Q 17.1. What is the difference between authentication and authorization?**

**Authentication** establishes who the caller is. **Authorization** decides whether that caller may do what the request asks. They are separate steps, done by separate middleware, and they fail with different status codes: 401 and 403 (section 18).

**Q 17.2. What are the building blocks?**

- A **claim** is one statement about the caller: a name, a role, an email, a permission.
- A **`ClaimsIdentity`** is a set of claims from one source. It counts as authenticated only if it has an authentication type.
- A **`ClaimsPrincipal`** is the user: one or more identities. It is available as `HttpContext.User`, or as a `ClaimsPrincipal` handler parameter.
- An **authentication scheme** is a name plus a **handler**. The handler knows one way of proving identity: a JWT in the `Authorization` header, a cookie, an API key.

**Q 17.3. What does an authentication handler do?**

Three things:

- **Authenticate:** read credentials from the request and return success (with a `ClaimsPrincipal`), failure, or "no result" when there are no credentials.
- **Challenge:** tell an unauthenticated caller how to authenticate. For an API that means 401, usually with a `WWW-Authenticate` header.
- **Forbid:** tell an authenticated caller the action is not allowed. For an API that means 403.

**Q 17.4. What does `UseAuthentication()` do, and what does it not do?**

It asks the default scheme's handler to authenticate, and sets `HttpContext.User` to the result. It does not reject anything. The output shows this: `/public` answers 200 with no key and with a valid key; only `isAuthenticated` differs. Rejection is the job of authorization, when an endpoint requires a user.

**Q 17.5. Which scheme should a real API use?**

- **JWT bearer tokens** issued by an identity provider (Microsoft Entra ID, Auth0, Keycloak and others) are the usual choice for APIs called by front ends, mobile apps and other services. Section 27 covers the setup.
- **Cookies** suit an API that serves only its own browser front end on the same site. On .NET 10, cookie authentication answers API endpoints with 401/403 and no longer redirects them to a login page.
- **API keys** are simple and common for server-to-server calls, but a key is a long-lived shared secret that identifies an application, not a user.

Do not write your own token format or password storage. The example writes a handler only to show how the parts connect.

### Example: `17-authentication.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// Register a SCHEME: a name ("ApiKey") tied to a handler that knows how to read credentials.
builder.Services
    .AddAuthentication(defaultScheme: "ApiKey")
    .AddScheme<AuthenticationSchemeOptions, ApiKeyHandler>("ApiKey", configureOptions: null);

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();   // runs the handler and fills HttpContext.User
app.UseAuthorization();    // enforces RequireAuthorization() / [Authorize]

// No authorization metadata: open to everyone. HttpContext.User still exists, but is anonymous.
app.MapGet("/public", (ClaimsPrincipal user) => new
{
    isAuthenticated = user.Identity?.IsAuthenticated ?? false,
});

// Requires an authenticated user.
app.MapGet("/me", (ClaimsPrincipal user) => new
{
    name = user.Identity!.Name,
    authenticationType = user.Identity.AuthenticationType,
    claims = user.Claims.Select(c => new { type = c.Type, value = c.Value }),
}).RequireAuthorization();

app.Run();

class ApiKeyHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    // Demo data. A real application looks keys up in a store and compares hashes.
    static readonly Dictionary<string, (string Name, string Role)> Keys = new()
    {
        ["key-alice"] = ("alice", "admin"),
        ["key-bob"] = ("bob", "reader"),
    };

    // "Who is this?"  Called once per request by UseAuthentication.
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Api-Key", out var apiKey))
        {
            return Task.FromResult(AuthenticateResult.NoResult());        // no credentials: anonymous
        }

        if (!Keys.TryGetValue(apiKey.ToString(), out var account))
        {
            return Task.FromResult(AuthenticateResult.Fail("Unknown API key"));
        }

        Claim[] claims =
        [
            new Claim(ClaimTypes.Name, account.Name),
            new Claim(ClaimTypes.Role, account.Role),
        ];
        var identity = new ClaimsIdentity(claims, authenticationType: Scheme.Name);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme.Name);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }

    // "Tell the caller how to authenticate."  Called when authorization needs a user and has none.
    protected override Task HandleChallengeAsync(AuthenticationProperties properties)
    {
        Response.StatusCode = StatusCodes.Status401Unauthorized;
        Response.Headers.WWWAuthenticate = "ApiKey";
        return Task.CompletedTask;
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 17-authentication.cs

$ curl -s http://localhost:5000/public
{"isAuthenticated":false}

$ curl -si http://localhost:5000/me
HTTP/1.1 401 Unauthorized
Content-Length: 0
WWW-Authenticate: ApiKey


$ curl -si http://localhost:5000/me -H "X-Api-Key: wrong"
HTTP/1.1 401 Unauthorized
Content-Length: 0
WWW-Authenticate: ApiKey


$ curl -s http://localhost:5000/me -H "X-Api-Key: key-alice"
{"name":"alice","authenticationType":"ApiKey","claims":[{"type":"http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name","value":"alice"},{"type":"http://schemas.microsoft.com/ws/2008/06/identity/claims/role","value":"admin"}]}

$ curl -s http://localhost:5000/public -H "X-Api-Key: key-alice"
{"isAuthenticated":true}
```

**What to notice**

- No key and a wrong key both end in 401 on `/me`. "No result" and "fail" are different answers from the handler, but the caller sees the same challenge.
- The 401 carries `WWW-Authenticate: ApiKey`, written by `HandleChallengeAsync`.
- The claim types are long URIs. `ClaimTypes.Name` and `ClaimTypes.Role` are constants for them. `user.Identity.Name` and role checks look for those types by default.

---

## 18. Authorization

**Builds on:** 17.

**Q 18.1. How do I require authorization on an endpoint?**

Minimal APIs: `.RequireAuthorization()` on an endpoint or group. Controllers: `[Authorize]` on an action or controller. With no arguments both apply the **default policy**, which requires an authenticated user. Pass a policy name to require more. `.AllowAnonymous()` / `[AllowAnonymous]` removes the requirement.

**Q 18.2. When is the answer 401 and when 403?**

- **401 Unauthorized:** the request has no valid credentials. The caller should authenticate and retry.
- **403 Forbidden:** the caller is known but is not allowed. Retrying with the same credentials will not help.

In the output, `/profile` with no user is 401, and `/admin` as `bob` (a reader) is 403.

**Q 18.3. What is a policy?**

A named set of requirements, registered once and applied by name. Built-in requirement helpers:

- `RequireAuthenticatedUser()`
- `RequireRole("admin")`
- `RequireClaim("permission", "refunds:write")`
- `RequireAssertion(context => ...)` for a small inline rule

For anything larger, write a **requirement** (a data class implementing `IAuthorizationRequirement`) and a **handler** (`AuthorizationHandler<TRequirement>`) that calls `context.Succeed(requirement)` when it is met. Handlers are resolved from DI, so they can use services.

Prefer policies named after what is being protected (`CanRefund`) over role names scattered through the code. When the rules change, you edit one policy.

**Q 18.4. What is the fallback policy?**

The policy applied to endpoints that have **no** authorization metadata at all. By default there is none, so an endpoint you forget to protect is public. Setting it to "authenticated user" reverses that: everything is protected unless marked `AllowAnonymous`.

One consequence to know: the fallback policy also applies when no endpoint matched. The last request below asks for a route that does not exist and gets **401**, not 404. Anonymous endpoints such as health checks (section 24) must opt out explicitly.

**Q 18.5. How do I authorize against the data itself ("only the owner may read this")?**

That decision needs the resource, which is loaded inside the handler, so it cannot be made by an attribute. Inject `IAuthorizationService`, load the resource, and call `AuthorizeAsync(user, resource, policyOrRequirements)`. Return `Results.Forbid()` (or 404, if you do not want to reveal that the resource exists) when it fails.

**Q 18.6. Where does `UseAuthorization()` go?**

After routing (it reads the selected endpoint's metadata), after `UseCors()`, after `UseAuthentication()`, and before the endpoints.

### Example: `18-authorization.cs`

The `X-User` header selects a demo user. `alice` is an admin with the refund permission and level 5. `bob` is a reader with level 1.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication("Demo")
    .AddScheme<AuthenticationSchemeOptions, DemoHandler>("Demo", null);

builder.Services.AddAuthorizationBuilder()
    // Secure by default: every endpoint needs an authenticated user unless it opts out.
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build())
    .AddPolicy("AdminOnly", policy => policy.RequireRole("admin"))
    .AddPolicy("CanRefund", policy => policy.RequireClaim("permission", "refunds:write"))
    .AddPolicy("Senior", policy => policy.AddRequirements(new MinimumLevelRequirement(3)));

builder.Services.AddSingleton<IAuthorizationHandler, MinimumLevelHandler>();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/public", () => "open to everyone").AllowAnonymous();
app.MapGet("/profile", (ClaimsPrincipal user) => $"hello {user.Identity!.Name}");   // fallback policy applies
app.MapGet("/admin", () => "admin area").RequireAuthorization("AdminOnly");
app.MapPost("/refunds", () => "refund issued").RequireAuthorization("CanRefund");
app.MapGet("/senior", () => "senior staff only").RequireAuthorization("Senior");

// Resource-based authorization: the decision needs the data, so it is made inside the handler.
app.MapGet("/documents/{id:int}", async (int id, ClaimsPrincipal user, IAuthorizationService authorization) =>
{
    var document = new Document(id, Owner: id == 1 ? "alice" : "bob");
    AuthorizationResult result = await authorization.AuthorizeAsync(
        user, document, new AuthorizationPolicyBuilder()
            .RequireAssertion(context => context.Resource is Document d && d.Owner == context.User.Identity!.Name)
            .Build());
    return result.Succeeded ? Results.Ok(document) : Results.Forbid();
});

app.Run();

record Document(int Id, string Owner);

// A custom requirement is plain data. Its handler holds the logic and can use DI.
record MinimumLevelRequirement(int Level) : IAuthorizationRequirement;

class MinimumLevelHandler : AuthorizationHandler<MinimumLevelRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinimumLevelRequirement requirement)
    {
        if (int.TryParse(context.User.FindFirstValue("level"), out int level) && level >= requirement.Level)
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;   // not calling Succeed = requirement not met
    }
}

// Demo authentication: the X-User header names a user from this table. Never do this for real.
class DemoHandler(IOptionsMonitor<AuthenticationSchemeOptions> options, ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    static readonly Dictionary<string, Claim[]> Users = new()
    {
        ["alice"] = [new(ClaimTypes.Name, "alice"), new(ClaimTypes.Role, "admin"),
                     new("permission", "refunds:write"), new("level", "5")],
        ["bob"] = [new(ClaimTypes.Name, "bob"), new(ClaimTypes.Role, "reader"), new("level", "1")],
    };

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-User", out var name) || !Users.TryGetValue(name.ToString(), out var claims))
        {
            return Task.FromResult(AuthenticateResult.NoResult());
        }

        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, Scheme.Name));
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name)));
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 18-authorization.cs

$ curl -s http://localhost:5000/public
open to everyone

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/profile
401

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/profile -H "X-User: bob"
200

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/admin -H "X-User: bob"
403

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/admin -H "X-User: alice"
200

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/refunds -H "X-User: bob"
403

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/refunds -H "X-User: alice"
200

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/senior -H "X-User: bob"
403

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/senior -H "X-User: alice"
200

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/documents/1 -H "X-User: alice"
200

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/documents/2 -H "X-User: alice"
403

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/no-such-route
401
```

**What to notice**

- `/profile` has no authorization call in the code. The fallback policy protected it.
- Role, claim and custom-requirement policies all produce 403 for `bob` and 200 for `alice`.
- `alice` can read document 1, which she owns, and gets 403 for document 2.

---

## 19. CORS

**Builds on:** 6, 7.

**Q 19.1. What problem does CORS solve?**

Browsers enforce the **same-origin policy**: JavaScript loaded from one origin may not read responses from another. An origin is scheme + host + port, so `https://app.example.com` and `https://api.example.com` are different origins. **Cross-Origin Resource Sharing** is how a server tells the browser which other origins may read its responses. It does so with response headers such as `Access-Control-Allow-Origin`.

**Q 19.2. Is CORS a security feature of my API?**

No. It relaxes a browser restriction. The server does not block anything, and non-browser clients ignore it. The third request below proves it: with a disallowed `Origin`, the server still returns 200 and the full body. The only difference is the missing `Access-Control-Allow-Origin` header, which makes a *browser* refuse to hand the response to the script. Protect data with authentication and authorization.

**Q 19.3. What is a preflight request?**

For requests that are not "simple" (methods other than GET/HEAD/POST, a JSON content type, custom headers such as `Authorization`), the browser first sends an `OPTIONS` request asking whether the real request is allowed. It includes `Access-Control-Request-Method` and `Access-Control-Request-Headers`. The CORS middleware answers it; your endpoint is not called.

In the output, the preflight gets 204 with the policy's allowed methods and headers. I also sent a preflight for `DELETE`, which the policy does not allow: the response still lists `GET,POST`. The browser compares that list with the method it wants and blocks the request itself.

**Q 19.4. How do I configure it safely?**

- List exact origins with `WithOrigins(...)`: scheme, host and port, with no path and no trailing slash.
- `AllowAnyOrigin()` is acceptable only for public, unauthenticated data.
- A wildcard origin cannot be combined with credentials. Building such a policy threw `InvalidOperationException: The CORS protocol does not allow specifying a wildcard (any) origin and credentials at the same time.` when I tried it.
- Allow only the methods and headers the front end uses.

**Q 19.5. Where does `UseCors()` go?**

After routing and before authentication, authorization and any response caching. A preflight request carries no credentials, so it must be answered before authorization runs. You can also apply policies per endpoint with `.RequireCors("name")` or `[EnableCors("name")]`.

### Example: `19-cors.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddPolicy("spa", policy => policy
        .WithOrigins("https://app.example.com")          // exact scheme + host + port, no trailing slash
        .WithMethods("GET", "POST")
        .WithHeaders("Content-Type", "Authorization"));
});

var app = builder.Build();

app.UseCors("spa");   // apply the policy to every endpoint

app.MapGet("/data", () => new { value = 42 });
app.MapPost("/data", () => Results.Created());

app.Run();
```

### Run it and verify

```text
# terminal 1:  dotnet run 19-cors.cs

$ curl -si http://localhost:5000/data
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"value":42}

$ curl -si http://localhost:5000/data -H "Origin: https://app.example.com"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Access-Control-Allow-Origin: https://app.example.com

{"value":42}

$ curl -si http://localhost:5000/data -H "Origin: https://evil.example.net"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"value":42}

$ curl -si -X OPTIONS http://localhost:5000/data -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: POST" -H "Access-Control-Request-Headers: content-type"
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type,Authorization
Access-Control-Allow-Methods: GET,POST
Access-Control-Allow-Origin: https://app.example.com


$ curl -si -X OPTIONS http://localhost:5000/data -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: DELETE"
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type,Authorization
Access-Control-Allow-Methods: GET,POST
Access-Control-Allow-Origin: https://app.example.com
```

---

## 20. Rate limiting

**Builds on:** 6, 7.

**Q 20.1. Why rate-limit an API?**

To keep one client, a bug in a retry loop, or an attack from consuming the capacity everyone shares, and to enforce usage tiers. ASP.NET Core has had rate-limiting middleware built in since .NET 7.

**Q 20.2. Which algorithms are available?**

| Limiter | Rule |
|---|---|
| Fixed window | N requests per time window; the counter resets when the window ends |
| Sliding window | N requests per window, with the window moving in segments, which smooths bursts at window boundaries |
| Token bucket | Requests spend tokens; tokens refill at a steady rate; allows short bursts up to the bucket size |
| Concurrency | At most N requests in progress at once |

**Q 20.3. Who is being limited?**

That depends on the **partition key**. A limiter without partitions is one counter for everybody, so one busy client exhausts it for all. A **partitioned** limiter keeps a counter per key: the authenticated user id, the API key or client id, or the IP address for anonymous traffic. The example partitions by an `X-Client-Id` header, and the output shows client `app-b` unaffected after `app-a` is blocked.

**Q 20.4. What does a rejected client receive?**

By default **503**. I read the default from `RateLimiterOptions.RejectionStatusCode`. Set it to **429 Too Many Requests**, and add a `Retry-After` header in `OnRejected` so well-behaved clients know when to try again.

`QueueLimit` lets requests wait for a permit instead of being rejected. Zero means reject immediately.

**Q 20.5. What are the limits of the built-in limiter?**

Counters are in the memory of one process. With several instances behind a load balancer, each instance counts separately, so the effective limit is multiplied. For a shared limit, enforce it at the gateway or use a distributed store.

**Q 20.6. Where does `UseRateLimiter()` go?**

After routing when you attach policies to endpoints with `RequireRateLimiting` or `[EnableRateLimiting]`, because the middleware has to know which endpoint was selected.

### Example: `20-rate-limiting.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;   // the default is 503

    // A partitioned policy: every distinct client id gets its own counter.
    options.AddPolicy("per-client", httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.Request.Headers["X-Client-Id"].ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 3,                       // 3 requests...
                Window = TimeSpan.FromSeconds(30),     // ...per 30-second window
                QueueLimit = 0,                        // reject immediately instead of queueing
            }));

    // Tell rejected clients when to come back.
    options.OnRejected = (context, cancellationToken) =>
    {
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out TimeSpan retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = ((int)Math.Ceiling(retryAfter.TotalSeconds)).ToString();
        }
        return ValueTask.CompletedTask;
    };
});

var app = builder.Build();

app.UseRateLimiter();

app.MapGet("/limited", () => "ok").RequireRateLimiting("per-client");
app.MapGet("/unlimited", () => "ok");

app.Run();
```

### Run it and verify

```text
# terminal 1:  dotnet run 20-rate-limiting.cs

$ for i in 1 2 3 4 5; do curl -s -o /dev/null -w "%{http_code} " http://localhost:5000/limited -H "X-Client-Id: app-a"; done; echo
200 200 200 429 429 

$ curl -si http://localhost:5000/limited -H "X-Client-Id: app-a" | sed "s/^Retry-After: .*/Retry-After: <seconds left in the window>/"
HTTP/1.1 429 Too Many Requests
Content-Length: 0
Retry-After: <seconds left in the window>


$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/limited -H "X-Client-Id: app-b"
200

$ for i in 1 2 3 4 5; do curl -s -o /dev/null -w "%{http_code} " http://localhost:5000/unlimited; done; echo
200 200 200 200 200 
```

**What to notice**

- Requests 1–3 from `app-a` pass. Requests 4 and 5 get 429.
- In the second block I replaced the `Retry-After` value with a placeholder, because it is the number of seconds left in the window and changes from run to run.
- `/unlimited` has no policy and is never limited.

---

# Part 6: Performance and integration

## 21. Caching

**Builds on:** 6, 9.

**Q 21.1. What kinds of caching does an HTTP API have?**

Three, and they can be combined:

1. **Client and proxy caching.** The response's `Cache-Control` header tells browsers and CDNs how long they may reuse it. The server does no work for a reused response and never sees the request.
2. **Conditional requests.** The server labels a response with an `ETag`. The client sends it back in `If-None-Match`. If nothing changed, the server answers **304 Not Modified** with no body. The request still happens, but the payload is not sent.
3. **Server-side output caching.** The server stores a rendered response and replays it for matching requests, without running the endpoint.

**Q 21.2. What does output caching do by default?**

Output caching (.NET 7+) is added with `AddOutputCache()` and `UseOutputCache()`, then enabled per endpoint with `.CacheOutput()`. The documented default policy:

- Only **200** responses to **GET** and **HEAD** are cached.
- Responses that set cookies are not cached.
- Responses to **authenticated** requests are not cached.
- The cache key is the full URL, query string included.
- Entries expire after **60 seconds** (I confirmed `OutputCacheOptions.DefaultExpirationTimeSpan` is one minute).

The store is in process memory by default, so each instance has its own. A Redis-backed store is available as a separate package.

**Q 21.3. How is that different from the older response caching middleware?**

Response caching obeys HTTP caching headers, including the client's. A client can send `Cache-Control: no-cache` and bypass it. Output caching is controlled by the server's policies, and entries can be removed on demand by tag. For an API, output caching is normally what you want.

**Q 21.4. How do I keep a cache from serving stale data?**

- Give entries a **tag** and call `IOutputCacheStore.EvictByTagAsync` when the underlying data changes. The example does this.
- Keep expiry times short for data that changes.
- Never cache user-specific data under a key that does not include the user. The default policy avoids this by skipping authenticated requests; be careful when you override it.

**Q 21.5. How does a conditional GET work?**

1. First response: `200`, body, `ETag: "v1"`.
2. Next request: `If-None-Match: "v1"`.
3. If the current version is still `v1`: `304`, no body. Otherwise `200`, the new body, a new ETag.

The ETag must change whenever the representation changes: a version number, a row version, or a hash of the content. Section 25 uses the same header for the opposite purpose, protecting writes.

**Q 21.6. Where does `UseOutputCache()` go?**

After `UseCors()`, after routing, and after authentication and authorization.

### Example: `21-caching.cs`

The `/report` handler counts its own executions, so a cached response is recognizable.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.OutputCaching;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOutputCache();     // server-side response cache (in memory by default)
var app = builder.Build();
app.UseOutputCache();

int executions = 0;
var product = new Product(1, "Hammer", Version: 1);

// 1. OUTPUT CACHING: the server stores the response and replays it without running the handler.
app.MapGet("/report", () => $"handler executed {Interlocked.Increment(ref executions)} time(s)")
   .CacheOutput(policy => policy.Expire(TimeSpan.FromSeconds(60)).Tag("reports"));

// Evict cached entries by tag when the underlying data changes.
app.MapPost("/report/invalidate", async (IOutputCacheStore cache, CancellationToken ct) =>
{
    await cache.EvictByTagAsync("reports", ct);
    return Results.NoContent();
});

// 2. CLIENT CACHING: the Cache-Control header lets browsers and proxies keep the response.
app.MapGet("/config", (HttpContext context) =>
{
    context.Response.Headers.CacheControl = "public, max-age=300";
    return new { theme = "dark" };
});

// 3. CONDITIONAL GET: an ETag lets the client ask "has it changed?" and get 304 with no body.
app.MapGet("/products/1", (HttpContext context) =>
{
    string etag = $"\"v{product.Version}\"";
    if (context.Request.Headers.IfNoneMatch == etag)
    {
        return Results.StatusCode(StatusCodes.Status304NotModified);
    }

    context.Response.Headers.ETag = etag;
    return Results.Ok(product);
});

app.MapPut("/products/1", (ProductUpdate update) =>
{
    product = product with { Name = update.Name, Version = product.Version + 1 };
    return Results.NoContent();
});

app.Run();

record Product(int Id, string Name, int Version);
record ProductUpdate(string Name);
```

### Run it and verify

```text
# terminal 1:  dotnet run 21-caching.cs

$ curl -s http://localhost:5000/report
handler executed 1 time(s)

$ curl -si http://localhost:5000/report | sed "s/^Age: .*/Age: <seconds since it was cached>/"
HTTP/1.1 200 OK
Content-Length: 26
Content-Type: text/plain; charset=utf-8
Age: <seconds since it was cached>

handler executed 1 time(s)

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/report/invalidate
204

$ curl -s http://localhost:5000/report
handler executed 2 time(s)

$ curl -si http://localhost:5000/config
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=300

{"theme":"dark"}

$ curl -si http://localhost:5000/products/1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
ETag: "v1"

{"id":1,"name":"Hammer","version":1}

$ curl -si http://localhost:5000/products/1 -H 'If-None-Match: "v1"'
HTTP/1.1 304 Not Modified


$ curl -s -o /dev/null -w "%{http_code}\n" -X PUT http://localhost:5000/products/1 -H "Content-Type: application/json" -d '{"name":"Claw hammer"}'
204

$ curl -si http://localhost:5000/products/1 -H 'If-None-Match: "v1"'
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
ETag: "v2"

{"id":1,"name":"Claw hammer","version":2}
```

**What to notice**

- The second `/report` call returns "1 time(s)" again and has an `Age` header: the handler did not run. (I replaced the `Age` value with a placeholder; it is the entry's age in seconds.)
- After the eviction call, the handler runs again and the count is 2.
- The conditional GET with the current ETag returns 304 with no body. After the PUT, the same request returns 200, the new data and `ETag: "v2"`.

---

## 22. Calling other services with `IHttpClientFactory`

**Builds on:** 3, 16.

**Q 22.1. Why not just create an `HttpClient` when I need one?**

Both simple approaches have a known problem:

- **A new `HttpClient` per call**, disposed afterwards: each one opens its own connections, and closed sockets are not released immediately. Under load the machine runs out of them.
- **One static `HttpClient` forever**: its connections stay open indefinitely, so it does not notice DNS changes.

`IHttpClientFactory` gives you cheap `HttpClient` objects on top of pooled message handlers that are recycled periodically. The default handler lifetime is **2 minutes**, which I confirmed from `HttpClientFactoryOptions.HandlerLifetime`.

**Q 22.2. What are named and typed clients?**

- A **named client** is configuration stored under a string: `AddHttpClient("catalog", ...)`, retrieved with `factory.CreateClient("catalog")`.
- A **typed client** is a class of yours that takes `HttpClient` in its constructor: `AddHttpClient<CatalogClient>(...)`. The class holds the URLs, serialization and error handling for one remote service, and the rest of the application calls its methods.

Prefer typed clients. Each is registered as **transient**. Do not store one in a singleton: the singleton would hold the same `HttpClient`, and its handler, for the life of the application, which defeats the recycling.

**Q 22.3. What is a `DelegatingHandler`?**

Middleware for outgoing requests. It wraps every call made by the client it is attached to and can add headers, log, measure or retry. The example adds a correlation id, and the "remote" endpoint reports that it received it.

**Q 22.4. How should my API react when the other service fails?**

- Set a **timeout** on the client and pass the request's `CancellationToken` to every call.
- Check the status code. `EnsureSuccessStatusCode()` throws for anything that is not 2xx; handle the statuses you expect (such as 404) before calling it.
- Do not pass the upstream error through unchanged. Translate it into your own contract: your own 404, or 502/503/504 when the dependency is the problem.
- For retries, circuit breakers and per-attempt timeouts, Microsoft documents the `Microsoft.Extensions.Http.Resilience` package and its `AddStandardResilienceHandler()`. I did not run it. Retry only operations that are idempotent (section 1).

### Example: `22-httpclient.cs`

To stay in one file, the app calls an endpoint of its own that plays the remote service.

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTransient<CorrelationHandler>();

// A TYPED client: CatalogClient receives an HttpClient configured here and managed by the factory.
builder.Services
    .AddHttpClient<CatalogClient>(client =>
    {
        client.BaseAddress = new Uri("http://localhost:5000");   // in real life: another service
        client.Timeout = TimeSpan.FromSeconds(5);
    })
    .AddHttpMessageHandler<CorrelationHandler>();                // outgoing "middleware"

var app = builder.Build();

// Pretend this endpoint belongs to a different service (the "catalog").
app.MapGet("/catalog/prices/{sku}", (string sku, HttpRequest request) =>
    sku == "missing"
        ? Results.NotFound()
        : Results.Ok(new Price(sku, 19.99m, SeenCorrelationId: request.Headers["X-Correlation-Id"].ToString())));

// Our API calls the catalog through the typed client.
app.MapGet("/orders/{sku}", async (string sku, CatalogClient catalog, CancellationToken ct) =>
{
    Price? price = await catalog.GetPriceAsync(sku, ct);
    return price is null
        ? Results.Problem($"No price for '{sku}'", statusCode: StatusCodes.Status404NotFound)
        : Results.Ok(new { sku, total = price.Amount * 2, price.SeenCorrelationId });
});

app.Run();

record Price(string Sku, decimal Amount, string SeenCorrelationId);

class CatalogClient(HttpClient http)
{
    public async Task<Price?> GetPriceAsync(string sku, CancellationToken ct)
    {
        using HttpResponseMessage response = await http.GetAsync($"/catalog/prices/{sku}", ct);
        if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return null;
        }

        response.EnsureSuccessStatusCode();   // any other non-2xx becomes an exception
        return await response.Content.ReadFromJsonAsync<Price>(ct);
    }
}

// A DelegatingHandler wraps every outgoing request made by the client it is attached to.
class CorrelationHandler : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        request.Headers.Add("X-Correlation-Id", "corr-123");
        return base.SendAsync(request, ct);
    }
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 22-httpclient.cs

$ curl -s http://localhost:5000/orders/A-100
{"sku":"A-100","total":39.98,"seenCorrelationId":"corr-123"}

$ curl -si http://localhost:5000/orders/missing
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.5","title":"Not Found","status":404,"detail":"No price for 'missing'"}

--- what the app logged for the outgoing calls ---
info: System.Net.Http.HttpClient.CatalogClient.LogicalHandler[100]
      Start processing HTTP request GET http://localhost:5000/catalog/prices/A-100
info: System.Net.Http.HttpClient.CatalogClient.ClientHandler[100]
      Sending HTTP request GET http://localhost:5000/catalog/prices/A-100
info: System.Net.Http.HttpClient.CatalogClient.ClientHandler[101]
      Received HTTP response headers after 64.3671ms - 200
info: System.Net.Http.HttpClient.CatalogClient.LogicalHandler[101]
      End processing HTTP request after 76.5432ms - 200
info: System.Net.Http.HttpClient.CatalogClient.LogicalHandler[100]
      Start processing HTTP request GET http://localhost:5000/catalog/prices/missing
info: System.Net.Http.HttpClient.CatalogClient.ClientHandler[100]
      Sending HTTP request GET http://localhost:5000/catalog/prices/missing
info: System.Net.Http.HttpClient.CatalogClient.ClientHandler[101]
      Received HTTP response headers after 2.5141ms - 404
info: System.Net.Http.HttpClient.CatalogClient.LogicalHandler[101]
      End processing HTTP request after 2.6321ms - 404
```

**What to notice**

- `seenCorrelationId` is `corr-123`: the handler's header reached the other endpoint.
- The upstream 404 became this API's own 404 problem-details response.
- The factory logs every outgoing request under a category that includes the client's name, with the status code and duration.

---

## 23. Background services

**Builds on:** 3, 16.

**Q 23.1. What is a hosted service?**

A class the host starts when the application starts and stops when it shuts down. Derive from `BackgroundService`, override `ExecuteAsync(CancellationToken stoppingToken)`, and register it with `AddHostedService<T>()`. Typical uses: draining a queue, periodic clean-up, warming a cache, consuming messages.

**Q 23.2. How does a background service use scoped services such as a database session?**

A hosted service is a singleton and there is no HTTP request, so no scope exists. Inject `IServiceScopeFactory`, create a scope for each unit of work, resolve what you need from it, and dispose it. This is the pattern from Q 3.4. Injecting a scoped service into the hosted service's constructor is a captive dependency.

**Q 23.3. What happens if `ExecuteAsync` throws?**

By default the host stops: `HostOptions.BackgroundServiceExceptionBehavior` is `StopHost`, which I confirmed. One bad item would therefore take the whole API down. Catch exceptions per item inside the loop, log them, and continue. The example processes a job that fails, and the jobs after it still complete.

**Q 23.4. How do I hand work from a request to the background?**

Put it on a queue the worker reads, and return **202 Accepted**, ideally with a URL where the client can check progress. `System.Threading.Channels` provides an in-process queue. A bounded channel applies backpressure: when it is full, writers wait.

An in-memory queue is lost when the process stops. For work that must not be lost, use a durable queue: a message broker, or a database table the worker polls.

**Q 23.5. How does shutdown work?**

`stoppingToken` is cancelled when shutdown begins. Pass it to every await so the loop ends promptly. The host waits up to `ShutdownTimeout` (30 seconds by default) for hosted services to stop.

### Example: `23-background.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Collections.Concurrent;
using System.Threading.Channels;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton(Channel.CreateBounded<string>(capacity: 100));   // in-memory queue
builder.Services.AddSingleton<CompletedJobs>();
builder.Services.AddScoped<JobProcessor>();          // scoped, like a DbContext would be
builder.Services.AddHostedService<QueueWorker>();    // started with the app, stopped with the app

var app = builder.Build();

// Accept the work, queue it, answer immediately with 202 Accepted.
app.MapPost("/jobs/{name}", async (string name, Channel<string> queue, CancellationToken ct) =>
{
    await queue.Writer.WriteAsync(name, ct);
    return Results.Accepted("/jobs");
});

app.MapGet("/jobs", (CompletedJobs completed) => completed.Items);

app.Run();

class QueueWorker(Channel<string> queue, IServiceScopeFactory scopeFactory, ILogger<QueueWorker> logger)
    : BackgroundService
{
    // Runs once for the lifetime of the application. stoppingToken fires at shutdown.
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Worker started");

        await foreach (string job in queue.Reader.ReadAllAsync(stoppingToken))
        {
            // A hosted service is a singleton. Scoped services need a scope created per unit of work.
            using IServiceScope scope = scopeFactory.CreateScope();
            var processor = scope.ServiceProvider.GetRequiredService<JobProcessor>();

            try
            {
                await processor.RunAsync(job, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                logger.LogError(ex, "Job {Job} failed", job);   // keep the loop alive
            }
        }
    }
}

class JobProcessor(CompletedJobs completed, ILogger<JobProcessor> logger)
{
    public async Task RunAsync(string job, CancellationToken ct)
    {
        await Task.Delay(TimeSpan.FromMilliseconds(200), ct);   // pretend to work
        if (job == "bad") throw new InvalidOperationException("cannot process this job");
        completed.Items.Enqueue(job);
        logger.LogInformation("Job {Job} done", job);
    }
}

class CompletedJobs
{
    public ConcurrentQueue<string> Items { get; } = new();
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 23-background.cs

$ curl -si -X POST http://localhost:5000/jobs/resize-images
HTTP/1.1 202 Accepted
Content-Length: 0
Location: /jobs


$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/jobs/bad
202

$ curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/jobs/send-emails
202

$ sleep 2; curl -s http://localhost:5000/jobs
["resize-images","send-emails"]

--- what the app logged ---
info: QueueWorker[0]
      Worker started
info: JobProcessor[0]
      Job resize-images done
fail: QueueWorker[0]
      Job bad failed
info: JobProcessor[0]
      Job send-emails done
```

**What to notice**

- All three POSTs returned 202 immediately, including the one whose job later failed. Accepting work and completing it are different things; a real API needs a way to report the outcome.
- The log shows the failure of `bad`, then `send-emails` completing after it.

---

# Part 7: Operating the API

## 24. Health checks

**Builds on:** 3, 7.

**Q 24.1. What are health checks for?**

Load balancers and orchestrators such as Kubernetes need a URL that says whether an instance is working. They call it every few seconds and act on the status code.

**Q 24.2. What is the difference between liveness and readiness?**

- **Liveness:** is the process running and able to respond? A failure means "restart me". It should check nothing external.
- **Readiness:** can this instance serve requests now? It checks what the instance depends on. A failure means "stop sending me traffic for now", not "restart me".

Keep dependency checks out of liveness. If the database is down, restarting every API instance does not help and makes recovery slower.

**Q 24.3. How are they built?**

`AddHealthChecks()` registers checks. A check is a lambda or a class implementing `IHealthCheck`, which gets its dependencies from DI. `MapHealthChecks(path, options)` exposes them. Tags plus a `Predicate` select which checks an endpoint runs.

**Q 24.4. How do results map to status codes?**

I read the defaults from `HealthCheckOptions.ResultStatusCodes`: `Healthy` → 200, `Degraded` → 200, `Unhealthy` → 503. The default body is the status as plain text. A `ResponseWriter` can produce JSON with per-check detail, as the readiness endpoint in the example does.

**Q 24.5. What should I watch for?**

- Checks run on every probe. Keep them fast, give them timeouts, and do not run expensive queries.
- A detailed body reveals your infrastructure. Expose it only internally.
- With a fallback authorization policy (Q 18.4), add `.AllowAnonymous()` to the health endpoints or the probes will get 401.

### Example: `24-health.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<FakeDatabase>();

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddCheck<DatabaseHealthCheck>("database", tags: ["ready"]);

var app = builder.Build();

// Liveness: "is the process running?"  Only the cheap check tagged "live".
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
});

// Readiness: "can it serve traffic?"  Dependency checks, with a JSON body listing each one.
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = (context, report) => context.Response.WriteAsJsonAsync(new
    {
        status = report.Status.ToString(),
        checks = report.Entries.Select(e => new
        {
            name = e.Key,
            status = e.Value.Status.ToString(),
            description = e.Value.Description,
        }),
    }),
});

// Test hook: break or repair the fake database.
app.MapPost("/toggle-database", (FakeDatabase database) =>
{
    database.IsUp = !database.IsUp;
    return new { database.IsUp };
});

app.Run();

class FakeDatabase
{
    public bool IsUp { get; set; } = true;
}

class DatabaseHealthCheck(FakeDatabase database) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken cancellationToken = default) =>
        Task.FromResult(database.IsUp
            ? HealthCheckResult.Healthy("Connection OK")
            : HealthCheckResult.Unhealthy("Cannot connect"));
}
```

### Run it and verify

```text
# terminal 1:  dotnet run 24-health.cs

$ curl -si http://localhost:5000/health/live
HTTP/1.1 200 OK
Content-Type: text/plain
Cache-Control: no-store, no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Pragma: no-cache

Healthy

$ curl -si http://localhost:5000/health/ready
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: no-store, no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Pragma: no-cache

{"status":"Healthy","checks":[{"name":"database","status":"Healthy","description":"Connection OK"}]}

$ curl -s -X POST http://localhost:5000/toggle-database
{"isUp":false}

$ curl -si http://localhost:5000/health/ready
HTTP/1.1 503 Service Unavailable
Content-Type: application/json; charset=utf-8
Cache-Control: no-store, no-cache
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Pragma: no-cache

{"status":"Unhealthy","checks":[{"name":"database","status":"Unhealthy","description":"Cannot connect"}]}

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/health/live
200
```

**What to notice**

- After the fake database is switched off, readiness is 503 while liveness stays 200.
- Health responses carry `Cache-Control: no-store, no-cache`, so no intermediary reuses an old status.

---

## 25. API design: paging, versioning, concurrency

**Builds on:** 9, 11, 21.

**Q 25.1. Why must list endpoints be paged?**

A list that returns everything works with 50 rows and fails with 5 million: memory, response time and client parsing all grow with the data. Page from the first version, because adding paging later breaks existing clients.

- **Offset paging** (`page`, `pageSize`) is simple and lets clients jump to any page. It gets slower for deep pages, and items can be skipped or repeated if rows are inserted while a client pages through.
- **Cursor (keyset) paging** ("give me 20 items after id 1234") stays fast and stable, but only supports "next".

Whichever you choose: default the page size, **cap** it on the server, and tell the client how to continue (a total count, or a next cursor/link). In the example a request for `pageSize=500` is served with 20.

**Q 25.2. How should filtering and sorting work?**

Through query-string parameters with fixed names: `?minPrice=430&sort=-price`. Accept only the fields you decide to support. Do not turn arbitrary client text into query expressions. Apply filter, sort and paging in the database query, not in memory after loading everything.

**Q 25.3. How do I change an API without breaking clients?**

Adding optional request fields, new response fields and new endpoints is normally safe. Removing or renaming fields, changing types or meaning, and making optional input required are breaking changes. Those need a new **version**, with the old one kept running until clients have moved.

Ways to carry the version:

| Where | Example | Notes |
|---|---|---|
| URL segment | `/api/v2/products` | Visible, easy to route and test. The most common choice. |
| Query string | `/api/products?api-version=2.0` | Keeps URLs stable |
| Header | `X-Api-Version: 2` | Clean URLs, less visible |
| Media type | `Accept: application/vnd.shop.v2+json` | Most faithful to HTTP, least convenient |

The example uses URL segments with one route group per version, which needs no library. The `Asp.Versioning.Http` and `Asp.Versioning.Mvc` NuGet packages add version negotiation, deprecation reporting and OpenAPI integration; I did not run them.

**Q 25.4. Two clients edit the same resource. How do I prevent the second from overwriting the first?**

With **optimistic concurrency**:

1. GET returns the resource with an `ETag` that identifies its version.
2. The client sends the update with `If-Match: <that ETag>`.
3. The server applies the update only if the current version still matches. Otherwise it returns **412 Precondition Failed**, and the client must read again and reapply its change.
4. If updates without `If-Match` are not acceptable, reject them with **428 Precondition Required**.

Without this the last writer silently wins. The check and the write must be atomic: the example uses `ConcurrentDictionary.TryUpdate`, and with a database you would use a row-version column so the `UPDATE` itself fails when the row has changed.

**Q 25.5. POST is not idempotent. How do I make retries safe?**

Let the client send a unique **idempotency key** (commonly a header named `Idempotency-Key`) with each logical operation. The server stores the key with the result. A repeated request with the same key returns the stored result and does not act again. Payment APIs work this way.

**Q 25.6. Which conventions keep an API predictable?**

- Plural nouns for collections: `/orders`, `/orders/42`, `/orders/42/items`.
- No verbs in paths when a method already says it. `DELETE /orders/42`, not `POST /orders/42/delete`.
- One casing style for JSON properties and one for URLs.
- One error format everywhere (section 14).
- UTC timestamps in ISO 8601.

### Example: `25-api-design.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using System.Collections.Concurrent;

var app = WebApplication.CreateBuilder(args).Build();

var products = new ConcurrentDictionary<int, Product>(
    Enumerable.Range(1, 45).ToDictionary(i => i, i => new Product(i, $"Product {i:00}", Price: i * 10m, Version: 1)));

// -- Versioning by URL segment: each version is a route group. ---------------
RouteGroupBuilder v1 = app.MapGroup("/api/v1/products");
RouteGroupBuilder v2 = app.MapGroup("/api/v2/products");

// v1: the original contract, a bare array, no paging.
v1.MapGet("/", () => products.Values.OrderBy(p => p.Id).Take(3).Select(p => new { p.Id, p.Name }));

// v2: paging, filtering and sorting, with the paging details in an envelope.
v2.MapGet("/", (HttpContext context, int page = 1, int pageSize = 10, decimal? minPrice = null, string sort = "id") =>
{
    page = Math.Max(page, 1);
    pageSize = Math.Clamp(pageSize, 1, 20);   // never let a client ask for everything

    IEnumerable<Product> query = products.Values.Where(p => minPrice is null || p.Price >= minPrice);
    query = sort == "-price" ? query.OrderByDescending(p => p.Price) : query.OrderBy(p => p.Id);

    int total = query.Count();
    var items = query.Skip((page - 1) * pageSize).Take(pageSize).Select(p => new { p.Id, p.Name, p.Price });

    context.Response.Headers["X-Total-Count"] = total.ToString();
    return new
    {
        page,
        pageSize,
        total,
        totalPages = (int)Math.Ceiling(total / (double)pageSize),
        items,
    };
});

// -- Optimistic concurrency: the client must prove it edited the latest version. --
v2.MapGet("/{id:int}", (int id, HttpContext context) =>
{
    if (!products.TryGetValue(id, out Product? product)) return Results.NotFound();
    context.Response.Headers.ETag = $"\"{product.Version}\"";
    return Results.Ok(product);
});

v2.MapPut("/{id:int}", (int id, ProductUpdate update, HttpContext context) =>
{
    if (!products.TryGetValue(id, out Product? current)) return Results.NotFound();

    string ifMatch = context.Request.Headers.IfMatch.ToString();
    if (ifMatch.Length == 0)
    {
        return Results.Problem("Send the ETag you read in an If-Match header.", statusCode: 428);
    }

    if (ifMatch != $"\"{current.Version}\"")
    {
        return Results.Problem("The product was changed by someone else. Read it again.", statusCode: 412);
    }

    Product updated = current with { Name = update.Name, Price = update.Price, Version = current.Version + 1 };
    // TryUpdate is atomic: it fails if another request replaced the value in the meantime.
    if (!products.TryUpdate(id, updated, current))
    {
        return Results.Problem("The product was changed by someone else. Read it again.", statusCode: 412);
    }

    context.Response.Headers.ETag = $"\"{updated.Version}\"";
    return Results.Ok(updated);
});

app.Run();

record Product(int Id, string Name, decimal Price, int Version);
record ProductUpdate(string Name, decimal Price);
```

### Run it and verify

```text
# terminal 1:  dotnet run 25-api-design.cs

$ curl -s http://localhost:5000/api/v1/products
[{"id":1,"name":"Product 01"},{"id":2,"name":"Product 02"},{"id":3,"name":"Product 03"}]

$ curl -si "http://localhost:5000/api/v2/products?page=2&pageSize=3"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Total-Count: 45

{"page":2,"pageSize":3,"total":45,"totalPages":15,"items":[{"id":4,"name":"Product 04","price":40},{"id":5,"name":"Product 05","price":50},{"id":6,"name":"Product 06","price":60}]}

$ curl -s "http://localhost:5000/api/v2/products?minPrice=430&sort=-price&pageSize=500"
{"page":1,"pageSize":20,"total":3,"totalPages":1,"items":[{"id":45,"name":"Product 45","price":450},{"id":44,"name":"Product 44","price":440},{"id":43,"name":"Product 43","price":430}]}

$ curl -si http://localhost:5000/api/v2/products/1
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
ETag: "1"

{"id":1,"name":"Product 01","price":10,"version":1}

$ curl -s -o /dev/null -w "%{http_code}\n" -X PUT http://localhost:5000/api/v2/products/1 -H "Content-Type: application/json" -d '{"name":"Renamed","price":12}'
428

$ curl -si -X PUT http://localhost:5000/api/v2/products/1 -H "Content-Type: application/json" -H 'If-Match: "1"' -d '{"name":"Renamed","price":12}'
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
ETag: "2"

{"id":1,"name":"Renamed","price":12,"version":2}

$ curl -si -X PUT http://localhost:5000/api/v2/products/1 -H "Content-Type: application/json" -H 'If-Match: "1"' -d '{"name":"Stale edit","price":99}'
HTTP/1.1 412 Precondition Failed
Content-Type: application/problem+json

{"type":"https://tools.ietf.org/html/rfc9110#section-15.5.13","title":"Precondition Failed","status":412,"detail":"The product was changed by someone else. Read it again."}
```

**What to notice**

- v1 and v2 return different shapes for the same resource, side by side.
- The first PUT has no `If-Match` and gets 428. The second sends `"1"` and succeeds; the resource moves to version 2. The third repeats `"1"`, which is now stale, and gets 412.

---

## 26. Hosting: reverse proxies, HTTPS and server limits

**Builds on:** 4, 6.

**Q 26.1. Why does a reverse proxy change what my app sees?**

In production Kestrel usually sits behind something: IIS, nginx, a cloud load balancer, an API gateway. The proxy accepts the client's HTTPS connection and makes its own, often plain HTTP, connection to your app. Your app then sees the proxy's IP address as the client and `http` as the scheme. That breaks generated URLs (the `Location` header from section 9), HTTPS redirection, IP-based rate limiting and audit logs.

Proxies pass the original values in the `X-Forwarded-For` and `X-Forwarded-Proto` headers. The **forwarded headers middleware** copies them back onto the request. It must run before anything that reads the scheme or client address.

**Q 26.2. Can a client fake those headers?**

A client can send them, so the middleware only honours them when the request comes from a trusted proxy. The defaults I read from `ForwardedHeadersOptions` trust loopback only (`KnownProxies` = `::1`, `KnownIPNetworks` = `127.0.0.0/8`) and process one proxy hop (`ForwardLimit` = 1). When the proxy is on another machine, add its address or network. In .NET 10 the networks property is `KnownIPNetworks`; the older `KnownNetworks` is obsolete.

The example works without extra configuration because curl connects from loopback.

**Q 26.3. How should HTTPS be enforced?**

- `app.UseHttpsRedirection()` redirects HTTP requests to HTTPS. The default status is 307.
- `app.UseHsts()` sends the `Strict-Transport-Security` header, telling browsers to use HTTPS only. The default max-age is 30 days and `localhost` is excluded. Use it outside Development.
- If TLS ends at the proxy, the redirect decision depends on the forwarded scheme, which is another reason to configure forwarded headers first.

I did not run these two because the examples listen on HTTP only. The default values are read from `HttpsRedirectionOptions` and `HstsOptions`.

**Q 26.4. Which server limits should I know?**

Kestrel rejects oversized or slow requests before your code runs. Defaults I read from `KestrelServerOptions.Limits`:

| Limit | Default |
|---|---|
| `MaxRequestBodySize` | 30,000,000 bytes (about 28.6 MB) |
| `KeepAliveTimeout` | 130 seconds |
| `RequestHeadersTimeout` | 30 seconds |
| `MaxConcurrentConnections` | unlimited |

A body over the limit gets **413**. The example lowers the limit to 1 KB to show it. If a proxy is in front, it has its own limits, and the smaller one applies.

**Q 26.5. How is the app configured in each environment?**

Through the configuration system from section 4, usually environment variables set by the platform: `ASPNETCORE_ENVIRONMENT`, `ASPNETCORE_URLS` (or `ASPNETCORE_HTTP_PORTS`), connection strings and secrets. The same build runs everywhere; only configuration differs.

### Example: `26-hosting.cs`

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property PublishAot=false

using Microsoft.AspNetCore.HttpOverrides;

var builder = WebApplication.CreateBuilder(args);

// Kestrel limits protect the server before your code runs.
builder.WebHost.ConfigureKestrel(kestrel =>
{
    kestrel.Limits.MaxRequestBodySize = 1024;   // 1 KB for the demo (the default is about 28.6 MB)
});

// Behind a reverse proxy the app sees the PROXY's address and scheme.
// Forwarded headers restore the original values the proxy passes along.
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    // By default only proxies on loopback are trusted. Add your proxy's address or network
    // to options.KnownProxies / options.KnownIPNetworks when it runs on another machine.
});

var app = builder.Build();

app.UseForwardedHeaders();   // must run before anything that looks at scheme or client IP

app.MapGet("/whoami", (HttpContext context) => new
{
    scheme = context.Request.Scheme,
    clientIp = context.Connection.RemoteIpAddress?.ToString(),
});

app.MapPost("/upload", async (HttpRequest request) =>
{
    using var reader = new StreamReader(request.Body);
    string body = await reader.ReadToEndAsync();
    return $"received {body.Length} characters";
});

app.Run();
```

### Run it and verify

```text
# terminal 1:  dotnet run 26-hosting.cs

$ curl -s http://localhost:5000/whoami
{"scheme":"http","clientIp":"127.0.0.1"}

$ curl -s http://localhost:5000/whoami -H "X-Forwarded-Proto: https" -H "X-Forwarded-For: 203.0.113.9"
{"scheme":"https","clientIp":"203.0.113.9"}

$ curl -s -X POST http://localhost:5000/upload -d "small body"
received 10 characters

$ head -c 5000 /dev/zero | tr "\0" "x" > big.txt; curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5000/upload --data-binary @big.txt
413
```

**What to notice**

- Without forwarded headers the app sees `http` and `127.0.0.1`. With them it sees `https` and `203.0.113.9`. (My test machine has no IPv6. Where `localhost` resolves to IPv6, the first address may be `::1`, which is also trusted by default.)
- The last command builds a 5,000-character file with Unix tools. On Windows, create any text file larger than 1 KB and send it with `--data-binary "@big.txt"`.
- The 5,000-character body was rejected with 413.

---

# Part 8: Topics that need NuGet packages

> **Read this first.** I did not execute the code in sections 27–30. Each needs a NuGet package, and nuget.org was not reachable from the machine I tested on. The commands and code follow Microsoft's documentation for ASP.NET Core 10, which I read on 4 October 2026 and link at the end. "Expected" in these sections means "what the documentation says", not "what I saw". Run them yourself before relying on them.
>
> These sections use normal projects, not single files.

## 27. JWT bearer authentication

**Builds on:** 17, 18. **Status: not executed.**

**Q 27.1. What is a JWT?**

A JSON Web Token is a string with three base64url parts separated by dots: a **header** (the signing algorithm), a **payload** (claims) and a **signature** over the first two. Standard claims include `iss` (issuer), `aud` (audience: the API the token is for), `exp` (expiry) and `sub` (subject: the user).

A JWT is signed, not encrypted. Anyone holding it can read the payload, so never put secrets in it. Nobody can change it without invalidating the signature.

**Q 27.2. What does the JWT bearer handler do?**

It is an authentication handler of the kind you saw in section 17. It reads the token from `Authorization: Bearer <token>`, checks the signature, issuer, audience and expiry, and on success builds the `ClaimsPrincipal` from the token's claims. Everything in section 18 then applies unchanged. Microsoft's documentation describes the outcomes the same way as section 18: 401 when the token is missing or invalid (bad signature, expired, wrong issuer or audience), 403 when the user is authenticated but lacks permission.

**Q 27.3. How is it configured?**

The package is `Microsoft.AspNetCore.Authentication.JwtBearer`. The documented basic configuration:

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(jwtOptions =>
    {
        jwtOptions.Authority = "https://{--your-authority--}";
        jwtOptions.Audience = "https://{--your-audience--}";
    });
```

`Authority` is your identity provider's address. The handler downloads the provider's signing keys from its metadata endpoint, so keys can rotate without redeploying the API. `Audience` is the identifier of this API; a token issued for a different API is rejected.

**Q 27.4. How do I try it locally without an identity provider?**

With the `dotnet user-jwts` tool, which the documentation describes as a tool for local development. These are the documented steps:

```bash
dotnet new web -o MyJWT
cd MyJWT
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

`Program.cs`:

```csharp
using System.Security.Claims;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthorization();
builder.Services.AddAuthentication("Bearer").AddJwtBearer();

var app = builder.Build();

app.UseAuthorization();

app.MapGet("/", () => "Hello, World!");
app.MapGet("/secret", (ClaimsPrincipal user) => $"Hello {user.Identity?.Name}. My secret")
    .RequireAuthorization();

app.Run();
```

Create a token and call the endpoint:

```bash
dotnet user-jwts create
dotnet run
curl -i -H "Authorization: Bearer {token}" https://localhost:{port}/secret
```

According to the documentation, `dotnet user-jwts create` prints a token and writes an `Authentication:Schemes:Bearer` section (valid audiences and the issuer `dotnet-user-jwts`) into `appsettings.Development.json`, which is why `AddJwtBearer()` needs no arguments here. Expected: `/secret` without the header returns 401, and with the token returns the greeting. Replace `{token}` and `{port}` with your values.

**Q 27.5. What are the common mistakes?**

- Turning validation off (`ValidateIssuer = false`, `ValidateAudience = false`, `ValidateLifetime = false`) to make an error go away.
- Long-lived access tokens. A JWT cannot be revoked easily, so keep lifetimes short and use refresh tokens.
- Being surprised by claim names. By default the handler renames some incoming claims to the long URI names you saw in section 17. `MapInboundClaims = false` keeps the original names such as `sub`.
- Sending tokens over plain HTTP.

---

## 28. Data access with EF Core

**Builds on:** 3, 11, 16. **Status: not executed.**

**Q 28.1. What is a `DbContext`, and what lifetime does it have?**

It represents a session with the database: it runs queries, tracks the entities it loaded, and writes their changes in `SaveChangesAsync`. `AddDbContext<T>()` registers it as **scoped**, so each request gets its own. That fits two facts: a context is not thread-safe, and its change tracker should not grow across requests. All of section 3 applies. In particular, a singleton or background service must create a scope to use one (Q 3.4, Q 23.2).

**Q 28.2. What does a minimal CRUD API on EF Core look like?**

This is Microsoft's tutorial code, using the in-memory provider so that no database server is needed:

```bash
dotnet new web -o TodoApi
cd TodoApi
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<TodoDb>(opt => opt.UseInMemoryDatabase("TodoList"));
var app = builder.Build();

app.MapGet("/todoitems", async (TodoDb db) =>
    await db.Todos.ToListAsync());

app.MapGet("/todoitems/{id}", async (int id, TodoDb db) =>
    await db.Todos.FindAsync(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound());

app.MapPost("/todoitems", async (Todo todo, TodoDb db) =>
{
    db.Todos.Add(todo);
    await db.SaveChangesAsync();

    return Results.Created($"/todoitems/{todo.Id}", todo);
});

app.MapPut("/todoitems/{id}", async (int id, Todo inputTodo, TodoDb db) =>
{
    var todo = await db.Todos.FindAsync(id);

    if (todo is null) return Results.NotFound();

    todo.Name = inputTodo.Name;
    todo.IsComplete = inputTodo.IsComplete;

    await db.SaveChangesAsync();

    return Results.NoContent();
});

app.MapDelete("/todoitems/{id}", async (int id, TodoDb db) =>
{
    if (await db.Todos.FindAsync(id) is Todo todo)
    {
        db.Todos.Remove(todo);
        await db.SaveChangesAsync();
        return Results.NoContent();
    }

    return Results.NotFound();
});

app.Run();

public class Todo
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public bool IsComplete { get; set; }
}

class TodoDb : DbContext
{
    public TodoDb(DbContextOptions<TodoDb> options)
        : base(options) { }

    public DbSet<Todo> Todos => Set<Todo>();
}
```

The handlers, status codes and DI are what you already know from section 11. `TodoDb` arrives as a handler parameter because it is a registered service.

**Q 28.3. What would I change before using that code for real?**

The tutorial keeps things short. Measured against earlier sections:

- It binds and returns the **entity**. Use request and response DTOs (Q 10.5).
- The list endpoint returns every row. Add paging, applied in the query with `Skip`/`Take` (Q 25.1).
- Pass the request's `CancellationToken` to `ToListAsync`, `FindAsync` and `SaveChangesAsync` (Q 16.3).
- For read-only queries add `AsNoTracking()`, and project to the DTO with `Select` so only the needed columns are read.
- Use a real provider (SQL Server, PostgreSQL, SQLite). The in-memory provider is for demos and simple tests; it does not behave like a relational database.
- Manage the schema with migrations (`dotnet ef migrations add`, `dotnet ef database update`).
- Add a concurrency token (a row-version column) for the optimistic concurrency from Q 25.4.

---

## 29. OpenAPI documents

**Builds on:** 9, 12. **Status: not executed** (except where noted).

**Q 29.1. What is an OpenAPI document?**

A machine-readable description of an API: its paths, methods, parameters, request and response bodies, and status codes. Tools use it to generate client code, interactive documentation, tests and gateway configuration.

**Q 29.2. How does ASP.NET Core produce one?**

With the `Microsoft.AspNetCore.OpenApi` package. The documented minimal setup:

```csharp
var builder = WebApplication.CreateBuilder();

builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.MapGet("/", () => "Hello world!");

app.Run();
```

The document is served at `/openapi/v1.json`. On .NET 10 it is generated as OpenAPI 3.1 by default, and YAML is available by mapping a route that ends in `.yaml`.

One part I did check: on my machine `dotnet new webapi` generated a `Program.cs` that already contains `AddOpenApi()` and the `MapOpenApi()` block above, with a reference to `Microsoft.AspNetCore.OpenApi` 10.0.12 in the project file. I could not restore or run it.

**Q 29.3. Is there a built-in documentation page (Swagger UI)?**

No. The documentation states that ASP.NET Core generates the document only, and that interactive UIs such as Swagger UI or Scalar must be added separately.

**Q 29.4. How does the generator know what my endpoints accept and return?**

From endpoint metadata:

- Parameter types and binding sources (section 8).
- Response types. `TypedResults` supplies them automatically; that is the metadata printed by the `/metadata` endpoint in section 9, which I did run. For `Results.*` or `IActionResult`, declare them with `.Produces<T>(statusCode)` or `[ProducesResponseType]`.
- Names and descriptions: `.WithName(...)`, `.WithSummary(...)`, `.WithDescription(...)`, `.WithTags(...)`.

This is the practical reason section 9 recommends `TypedResults`.

**Q 29.5. Should the document be public?**

The documented sample maps it only in Development. Whether to publish it is your decision: a public API usually does, an internal one usually does not.

---

## 30. Integration testing

**Builds on:** 2, 3. **Status: not executed** (except where noted).

**Q 30.1. What is an integration test for an API?**

A test that sends real HTTP requests through the real pipeline (routing, binding, validation, filters, authorization, serialization) and asserts on the response. A unit test of a handler method checks your logic. An integration test checks that the pieces are wired together correctly, which is where many API bugs are.

**Q 30.2. How does `WebApplicationFactory` work?**

`WebApplicationFactory<Program>` (package `Microsoft.AspNetCore.Mvc.Testing`) starts your application inside the test process on an in-memory test server. `CreateClient()` returns an `HttpClient` connected to it. No network port is opened.

Setup, assuming the API project from section 28 is in `TodoApi`:

```bash
dotnet new xunit -o TodoApi.Tests
cd TodoApi.Tests
dotnet add reference ../TodoApi/TodoApi.csproj
dotnet add package Microsoft.AspNetCore.Mvc.Testing
```

The documentation lists two requirements for the test project: reference that package, and use the Web SDK. For the second, open `TodoApi.Tests.csproj` and change its first line to `<Project Sdk="Microsoft.NET.Sdk.Web">`. (The xunit template creates it with `Microsoft.NET.Sdk`; I checked the template's output.)

A test, adapted from the documented pattern:

```csharp
using Microsoft.AspNetCore.Mvc.Testing;

public class TodoApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public TodoApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }

    [Fact]
    public async Task Get_todoitems_returns_success_and_json()
    {
        // Arrange
        var client = _factory.CreateClient();

        // Act
        var response = await client.GetAsync("/todoitems");

        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/json; charset=utf-8",
            response.Content.Headers.ContentType?.ToString());
    }
}
```

Run with `dotnet test`.

**Q 30.3. The test project cannot see `Program`. What do I do?**

`Program` must be visible to the test project. On .NET 10 it is: I printed `typeof(Program).IsPublic` in both a single-file app and a `dotnet new web` project and got `True`. On earlier versions the generated class is internal, and the documented fixes are to add `public partial class Program { }` at the end of `Program.cs`, or to add `<InternalsVisibleTo Include="YourTestProjectName" />` to the API's project file.

**Q 30.4. How do I replace a dependency in tests?**

Override the registration for the test host. The documented pattern:

```csharp
var client = _factory.WithWebHostBuilder(builder =>
    {
        builder.ConfigureTestServices(services =>
        {
            services.AddScoped<IQuoteService, TestQuoteService>();
        });
    })
    .CreateClient();
```

This is the payoff of depending on interfaces registered in DI (section 3): the test swaps the database, the clock or an external HTTP client and leaves everything else real.

**Q 30.5. What is worth testing this way?**

- Status codes and response shapes for the success path and each documented failure.
- Validation: an invalid body returns 400 with the expected error keys.
- Security: 401 without credentials, 403 with the wrong role.
- The error format: an unhandled exception returns problem details and leaks nothing.

---

# Appendix A: Recommended middleware order for an API

Combining Microsoft's ordering rules with the sections above:

```csharp
var app = builder.Build();

app.UseForwardedHeaders();     // 26  first, so scheme and client IP are right for everything below
app.UseExceptionHandler();     // 14  early, to catch exceptions from everything below
app.UseStatusCodePages();      // 14
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();             // 26
}
app.UseHttpsRedirection();     // 26

app.UseRouting();              //  7  optional: added automatically if omitted
app.UseRateLimiter();          // 20  after routing when endpoint policies are used
app.UseCors();                 // 19  before authentication and authorization
app.UseAuthentication();       // 17
app.UseAuthorization();        // 18
app.UseRequestTimeouts();      // 16  after routing
app.UseOutputCache();          // 21  after CORS, routing and authorization

app.MapHealthChecks("/health/live");   // 24
app.MapControllers();                  // 12  and/or app.MapGet(...), app.MapGroup(...)

app.Run();
```

Rules behind this order, from the documentation: CORS, authentication and authorization must appear in that sequence; the rate limiter must follow routing when endpoint-specific policies are used; request timeouts must follow routing; output caching goes after CORS, routing and authentication/authorization; forwarded headers go before anything that reads the scheme, host or client IP. I did not run this exact combination as one program.

# Appendix B: What was verified, and how

**Executed (sections 1–26).** All 26 examples were compiled and run as single-file apps with the .NET SDK 10.0.112 on Ubuntu 24.04, exactly as printed, and every command in the "Run it and verify" blocks was sent with curl. The output blocks were captured by a script and inserted into this document without retyping. Edits to the captured output are limited to: removing the `Date`, `Server` and `Transfer-Encoding` headers; replacing the `Retry-After` and `Age` values with placeholders; and reformatting the provider list in section 4 and the log entries in section 5 for readability, as noted there.

**Checked with separate small programs.** These claims are not shown by the examples, so I tested each on its own:

| Claim | Section |
|---|---|
| Without `PublishAot=false`, a single-file app fails on JSON with `NotSupportedException` | How to use |
| An example pasted into a `dotnet new web` project runs after removing the `#:` lines; leaving them in gives `CS9298` | How to use |
| A singleton depending on a scoped service fails at startup in Development and starts in Production | Q 3.3 |
| Scope validation is on in Development and off in Production by default | Q 3.3 |
| DI lifetimes of framework services | Q 3.6 |
| `<AppName>.settings.json` providers appear in a project-based app too | Q 4.1 |
| Contents of the trailing chained configuration provider; `appsettings.json` beats an `ASPNETCORE_`-prefixed variable for the same key | Section 4 notes |
| Authentication/authorization middleware is added automatically when omitted | Q 6.5 |
| Two identical routes cause `AmbiguousMatchException` and a 500 | Q 7.3 |
| A missing required parameter gives an empty 400 in Production and a text explanation in Development | Q 8.3 |
| A complex parameter on a GET fails with "Body was inferred..." | Q 8.4 |
| Minimal API and controller JSON options are independent | Q 10.2 |
| `required`, `RespectRequiredConstructorParameters` and `RespectNullableAnnotations` reject missing or null values | Q 10.4 |
| Minimal API validation output without `AddProblemDetails()` | Q 13.4 |
| An unhandled exception in Production gives an empty 500 | Q 14.1 |
| A wildcard CORS origin with credentials throws | Q 19.4 |
| `typeof(Program).IsPublic` is `True` on .NET 10 | Q 30.3 |
| `KnownNetworks` is obsolete in .NET 10 (compiler warning `ASPDEPR005`) | Q 26.2 |
| `dotnet new webapi` includes `AddOpenApi()`/`MapOpenApi()` | Q 29.2 |
| `WithName`, `WithSummary`, `WithDescription`, `WithTags` and `Produces<T>` compile against the shared framework | Q 29.4 |
| The xunit template's project file uses `Microsoft.NET.Sdk` | Q 30.2 |

**Read from the running framework.** Default values quoted in the text were printed from the option objects: Kestrel limits, `HostOptions.ShutdownTimeout` and `BackgroundServiceExceptionBehavior`, `HttpClientFactoryOptions.HandlerLifetime`, `ForwardedHeadersOptions`, `RateLimiterOptions.RejectionStatusCode`, `OutputCacheOptions.DefaultExpirationTimeSpan`, `LoggerFilterOptions.MinLevel`, `HstsOptions`, `HttpsRedirectionOptions`, `HealthCheckOptions.ResultStatusCodes`, `RouteOptions.LowercaseUrls`, the default and fallback authorization policies, and the JSON serializer defaults.

**Taken from Microsoft's documentation, not executed.**

- Sections 27–30 in full, apart from the items marked as checked.
- The default configuration source order (Q 4.1), the route precedence rules (Q 7.3), the `[ApiController]` behaviours (Q 12.2), the MVC filter kinds (Q 15.3), the request-timeout notes about debuggers and ordering (Q 16.4), the output-cache default policy other than its expiry (Q 21.2), and the middleware ordering rules (Appendix A).
- `UseHttpsRedirection` and `UseHsts` behaviour (Q 26.3): only their default option values were read.
- The `Microsoft.Extensions.Http.Resilience` and `Asp.Versioning.*` packages are mentioned as pointers only.

**Not tested at all.** Anything on .NET 8 or 9, on Windows or macOS, in Visual Studio, or under IIS.

# Sources

Microsoft Learn, ASP.NET Core 10 documentation, read 4 October 2026:

- [File-based apps](https://learn.microsoft.com/en-us/dotnet/core/sdk/file-based-apps)
- [What's new in ASP.NET Core in .NET 10](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0?view=aspnetcore-10.0)
- [APIs overview: Minimal APIs and controllers](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis?view=aspnetcore-10.0)
- [Configuration in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-10.0)
- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-10.0)
- [Create web APIs with ASP.NET Core (controllers)](https://learn.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-10.0)
- [Handle errors in ASP.NET Core APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling-api?view=aspnetcore-10.0)
- [Request timeouts middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/timeouts?view=aspnetcore-10.0)
- [Output caching middleware](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output?view=aspnetcore-10.0)
- [Make HTTP requests using IHttpClientFactory](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-10.0)
- [Build resilient HTTP apps](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)
- [Configure JWT bearer authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-jwt-bearer-authentication?view=aspnetcore-10.0)
- [Manage JSON Web Tokens in development with dotnet user-jwts](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn?view=aspnetcore-10.0)
- [Tutorial: Create a Minimal API with ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/tutorials/min-web-api?view=aspnetcore-10.0)
- [Generate OpenAPI documents](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/aspnetcore-openapi?view=aspnetcore-10.0)
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-10.0)
- [.NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
