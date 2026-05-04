# From Old Threading Habits to Modern .NET 8 APIs with TAP

When developers move from older .NET code to modern ASP.NET Core, the biggest shift is not just syntax. It is a change in **how work is modeled**: old code often blocks threads and mixes sync and async, while modern .NET 8 uses the Task-based Asynchronous Pattern (TAP) to keep request handling scalable and predictable. Microsoft recommends making controller actions asynchronous, keeping the entire call stack asynchronous, and avoiding blocking calls where async APIs exist.

This tutorial uses a simple rule throughout: first show the **old approach**, then explain the **issue**, then show the **modern approach**, then explain **how it fixes the issue**. That structure is closer to how real teams learn and refactor code.

## TAP in one sentence

TAP means asynchronous methods return `Task` or `Task<T>` and are consumed with `await`. It is the recommended pattern for new asynchronous work in .NET, and it is the foundation of modern ASP.NET Core, EF Core, and `HttpClient` APIs.

A good mental model is this:

```text
Old style:
Request thread -> waits -> waits -> waits -> returns

TAP style:
Request thread -> starts async work -> released
I/O completes -> continuation resumes -> returns
```

That is why async in web APIs is mainly about **scalability**, not raw CPU speed. A blocked thread cannot handle another request, but an awaited I/O operation lets the thread return to the pool.

## Example 1: Controller action

### Old approach

```csharp
[HttpGet("{id}")]
public IActionResult GetOrder(int id)
{
    var order = _orderService.GetOrder(id);
    return Ok(order);
}
```


### Issue

This blocks the request thread until the entire call finishes. In ASP.NET Core, too many blocking calls can lead to thread pool starvation and degraded response times under load.

### Modern approach

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id, CancellationToken cancellationToken)
{
    var order = await _orderService.GetOrderAsync(id, cancellationToken);
    return Ok(order);
}
```


### How it fixes it

The controller action is asynchronous, so while the database or external service is busy, the request thread is not unnecessarily blocked. Microsoft explicitly recommends making controller actions asynchronous and keeping the call stack async.

## Example 2: EF Core query

### Old approach

```csharp
public Order GetOrder(int id)
{
    return _db.Orders.First(o => o.Id == id);
}
```


### Issue

The thread waits synchronously for the database query to complete. EF Core documents async operations specifically because they avoid blocking a thread while the query runs in the database.

### Modern approach

```csharp
public async Task<Order> GetOrderAsync(int id, CancellationToken cancellationToken)
{
    return await _db.Orders.FirstAsync(o => o.Id == id, cancellationToken);
}
```


### How it fixes it

`FirstAsync` lets the framework use asynchronous I/O instead of tying up a worker thread while waiting for the database engine. That improves scalability for APIs handling many concurrent requests.

## Example 3: Saving to the database

### Old approach

```csharp
public Order CreateOrder(Order order)
{
    _db.Orders.Add(order);
    _db.SaveChanges();
    return order;
}
```


### Issue

`SaveChanges()` blocks while the database write completes. In web apps, repeated synchronous database writes reduce throughput because request threads sit idle waiting for I/O.

### Modern approach

```csharp
public async Task<Order> CreateOrderAsync(Order order, CancellationToken cancellationToken)
{
    await _db.Orders.AddAsync(order, cancellationToken);
    await _db.SaveChangesAsync(cancellationToken);
    return order;
}
```


### How it fixes it

`SaveChangesAsync` makes the write non-blocking from the request thread’s perspective. The app can handle more concurrent requests with the same thread pool.

## Example 4: External API calls

### Old approach

```csharp
public string GetPaymentStatus(string paymentId)
{
    var response = _httpClient.GetAsync($"/payments/{paymentId}").Result;
    return response.Content.ReadAsStringAsync().Result;
}
```


### Issue

This is classic sync-over-async. It blocks a thread on async APIs and works against how `HttpClient` is designed to be used. ASP.NET Core guidance says to avoid blocking calls and to keep I/O asynchronous.

### Modern approach

```csharp
public async Task<string> GetPaymentStatusAsync(
    string paymentId,
    CancellationToken cancellationToken)
{
    var response = await _httpClient.GetAsync($"/payments/{paymentId}", cancellationToken);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync(cancellationToken);
}
```


### How it fixes it

The outbound HTTP call is awaited asynchronously, so the request thread is free while the remote server responds. This is the intended usage pattern for `HttpClient` in modern .NET apps.

## Example 5: Sequential async when work could run in parallel

### Old approach

```csharp
public async Task<OrderSummaryDto> GetSummaryAsync(int orderId, CancellationToken cancellationToken)
{
    var order = await _db.Orders.FirstAsync(x => x.Id == orderId, cancellationToken);
    var customer = await _db.Customers.FirstAsync(x => x.Id == order.CustomerId, cancellationToken);
    var payment = await _paymentClient.GetPaymentStatusAsync(order.PaymentId, cancellationToken);

    return new OrderSummaryDto
    {
        OrderId = order.Id,
        CustomerName = customer.Name,
        PaymentStatus = payment
    };
}
```


### Issue

This is async, but not optimal. After loading the order, the customer lookup and payment lookup are independent, yet the code waits for one and only then starts the other. That adds unnecessary latency.

### Modern approach

```csharp
public async Task<OrderSummaryDto> GetSummaryAsync(int orderId, CancellationToken cancellationToken)
{
    var order = await _db.Orders.FirstAsync(x => x.Id == orderId, cancellationToken);

    var customerTask = _db.Customers.FirstAsync(x => x.Id == order.CustomerId, cancellationToken);
    var paymentTask = _paymentClient.GetPaymentStatusAsync(order.PaymentId, cancellationToken);

    await Task.WhenAll(customerTask, paymentTask);

    return new OrderSummaryDto
    {
        OrderId = order.Id,
        CustomerName = customerTask.Result.Name,
        PaymentStatus = paymentTask.Result
    };
}
```


### How it fixes it

`Task.WhenAll` lets independent I/O run concurrently, reducing total wait time. One important EF Core rule applies here: do not run multiple parallel operations on the same `DbContext` instance, so parallelization should be limited to safe independent operations, especially when one task is external HTTP and the other is database access handled appropriately.

## Example 6: No cancellation support

### Old approach

```csharp
public async Task<byte[]> ExportReportAsync()
{
    var data = await _db.Orders.ToListAsync();
    await Task.Delay(10000);
    return GenerateFile(data);
}
```


### Issue

If the client disconnects or the request times out, the server may continue doing expensive work for no reason. Modern .NET uses cooperative cancellation through `CancellationToken` to stop unnecessary operations.

### Modern approach

```csharp
public async Task<byte[]> ExportReportAsync(CancellationToken cancellationToken)
{
    var data = await _db.Orders.ToListAsync(cancellationToken);
    await Task.Delay(10000, cancellationToken);
    cancellationToken.ThrowIfCancellationRequested();

    return GenerateFile(data);
}
```

Controller:

```csharp
[HttpGet("export")]
public async Task<IActionResult> Export(CancellationToken cancellationToken)
{
    var file = await _reportService.ExportReportAsync(cancellationToken);
    return File(file, "application/pdf", "report.pdf");
}
```


### How it fixes it

The cancellation token flows from the HTTP request into the service and into EF Core. If the request ends early, the work can stop gracefully instead of wasting CPU, memory, and database activity.

## Example 7: Fire-and-forget background work inside a request

### Old approach

```csharp
[HttpPost]
public async Task<IActionResult> PlaceOrder(Order order)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync();

    Task.Run(() => _emailService.SendConfirmation(order.Email));

    return Ok();
}
```


### Issue

Using `Task.Run` as fire-and-forget work inside a web request is unsafe. ASP.NET Core best practices warn against patterns where work continues after the request lifecycle without proper hosting, tracking, or dependency scope handling. Hosted services are the framework-supported way to run background tasks.

### Modern approach

Queue:

```csharp
public class EmailQueue
{
    private readonly Channel<string> _channel = Channel.CreateUnbounded<string>();

    public ValueTask QueueAsync(string email, CancellationToken cancellationToken)
        => _channel.Writer.WriteAsync(email, cancellationToken);

    public IAsyncEnumerable<string> ReadAllAsync(CancellationToken cancellationToken)
        => _channel.Reader.ReadAllAsync(cancellationToken);
}
```

Worker:

```csharp
public class EmailWorker : BackgroundService
{
    private readonly EmailQueue _queue;
    private readonly EmailService _emailService;

    public EmailWorker(EmailQueue queue, EmailService emailService)
    {
        _queue = queue;
        _emailService = emailService;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var email in _queue.ReadAllAsync(stoppingToken))
        {
            await _emailService.SendConfirmationAsync(email, stoppingToken);
        }
    }
}
```

Usage:

```csharp
[HttpPost]
public async Task<IActionResult> PlaceOrder(Order order, CancellationToken cancellationToken)
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(cancellationToken);

    await _emailQueue.QueueAsync(order.Email, cancellationToken);

    return Ok();
}
```


### How it fixes it

The request only queues the email work and completes quickly, while the hosted worker handles delivery in a controlled background process. This is the recommended ASP.NET Core pattern for background tasks.

## Example 8: Repeating try-catch in every endpoint

### Old approach

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    try
    {
        var order = await _orderService.GetOrderAsync(id, CancellationToken.None);
        return Ok(order);
    }
    catch (Exception ex)
    {
        return StatusCode(500, ex.Message);
    }
}
```


### Issue

This duplicates error handling and tends to create inconsistent response shapes across endpoints. ASP.NET Core includes centralized error-handling mechanisms so business and controller code stay focused on normal flow.

### Modern approach

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;

        await httpContext.Response.WriteAsJsonAsync(new
        {
            error = exception.Message
        }, cancellationToken);

        return true;
    }
}
```

Registration:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```


### How it fixes it

Exceptions can bubble up naturally, and the platform converts them into consistent API responses in one place. That reduces repetition and keeps controllers cleaner.

## One complete workflow

Here is a single modern flow combining several TAP patterns:

```text
Client request
   |
   v
Async controller
   |
   v
Async service
   |-- EF Core query async
   |-- HttpClient call async
   |-- Task.WhenAll for independent work
   |-- SaveChangesAsync
   |-- Queue background email
   |
   v
Response returned

BackgroundService
   |
   v
Process queued email asynchronously
```

This workflow solves the main issues of old API code: blocking request threads, sync-over-async, duplicated exception handling, lack of cancellation, and unsafe background work. It also matches the async design of ASP.NET Core, EF Core, and `HttpClient`.

## Common mistakes to avoid

- Do not call `.Result` or `.Wait()` on async operations in APIs.
- Do not use `Task.Run()` just to make synchronous code “look async.” ASP.NET Core guidance explicitly warns against this.
- Do not use `async void` in ASP.NET Core app code.
- Do not mix sync and async carelessly; keep the whole call stack async where appropriate.
- Do not run multiple parallel operations on the same EF Core `DbContext`.


## The practical rule

If your API is doing database access, HTTP calls, file I/O, or any wait-heavy operation, use TAP with `Task`, `async`, and `await`. That is how modern .NET 8 APIs are built, and it is the reason those apps scale better than older thread-blocking designs.

