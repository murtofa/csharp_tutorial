
# Task-based Asynchronous Pattern (TAP) in Modern .NET 8 API Programming: Complete Use Cases with Examples

The Task-based Asynchronous Pattern (TAP) is Microsoft's **recommended approach for all new asynchronous development** in .NET 8. TAP uses a single `async` method returning `Task` or `Task<T>` to represent both the start and completion of an asynchronous operation.

Here's a comprehensive guide covering **every major use case** in modern .NET 8 API programming.

***

## 1. Database Operations with Entity Framework Core

### Use Case: Async CRUD Operations

TAP is essential for database operations because they are I/O-bound and benefit from non-blocking execution.

```csharp
public class ProductService
{
    private readonly AppDbContext _context;

    public ProductService(AppDbContext context)
    {
        _context = context;
    }

    // READ: Query with filtering
    public async Task<List<Product>> GetProductsAsync(string category)
    {
        return await _context.Products
            .Where(p => p.Category == category)
            .ToListAsync(); // EF Core async method
    }

    // READ: Single item
    public async Task<Product?> GetProductByIdAsync(int id)
    {
        return await _context.Products
            .FirstOrDefaultAsync(p => p.Id == id);
    }

    // CREATE
    public async Task<Product> CreateProductAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
        return product;
    }

    // UPDATE
    public async Task<Product?> UpdateProductAsync(int id, Product updatedProduct)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null) return null;

        product.Name = updatedProduct.Name;
        product.Price = updatedProduct.Price;
        
        await _context.SaveChangesAsync();
        return product;
    }

    // DELETE
    public async Task<bool> DeleteProductAsync(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null) return false;

        _context.Products.Remove(product);
        await _context.SaveChangesAsync();
        return true;
    }
}
```

**Key EF Core async methods**:

- `ToListAsync()`, `FirstOrDefaultAsync()`, `SingleOrDefaultAsync()`
- `FindAsync()`, `AddAsync()`, `SaveChangesAsync()`

**Why TAP here**: Database calls are I/O-bound. `await` frees up the thread while waiting for the database response, improving scalability.

***

## 2. ASP.NET Core 8 Web API Controllers

### Use Case: RESTful API Endpoints

Controllers in .NET 8 should return `Task<IActionResult>` or `Task<ActionResult<T>>` for all async operations.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService _productService;

    public ProductsController(ProductService productService)
    {
        _productService = productService;
    }

    // GET: api/products?category=Electronics
    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetProducts(
        [FromQuery] string category)
    {
        var products = await _productService.GetProductsAsync(category);
        return Ok(products);
    }

    // GET: api/products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetProduct(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        
        if (product == null)
            return NotFound();
            
        return Ok(product);
    }

    // POST: api/products
    [HttpPost]
    public async Task<ActionResult<Product>> CreateProduct(Product product)
    {
        var created = await _productService.CreateProductAsync(product);
        return CreatedAtAction(
            nameof(GetProduct), 
            new { id = created.Id }, 
            created);
    }

    // PUT: api/products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(int id, Product product)
    {
        var updated = await _productService.UpdateProductAsync(id, product);
        
        if (updated == null)
            return NotFound();
            
        return NoContent();
    }

    // DELETE: api/products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var deleted = await _productService.DeleteProductAsync(id);
        
        if (!deleted)
            return NotFound();
            
        return NoContent();
    }
}
```

**TAP controller rules**:

- Mark methods with `async` keyword

```
- Return `Task<IActionResult>` or `Task<ActionResult<T>>`
```

- Use `await` for all I/O operations

***

## 3. HTTP Client Calls (External APIs)

### Use Case: Calling External REST APIs

`HttpClient` is TAP-native and designed for async-first usage.

```csharp
public class WeatherService
{
    private readonly HttpClient _httpClient;

    public WeatherService(HttpClient httpClient)
    {
        _httpClient = httpClient;
        _httpClient.BaseAddress = new Uri("https://api.weather.com/");
    }

    // GET request
    public async Task<WeatherData?> GetWeatherAsync(string city)
    {
        var response = await _httpClient.GetAsync($"weather/{city}");
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<WeatherData>();
    }

    // POST request
    public async Task<bool> SubmitWeatherReportAsync(WeatherReport report)
    {
        var response = await _httpClient.PostAsJsonAsync("reports", report);
        return response.IsSuccessStatusCode;
    }

    // PUT request
    public async Task<bool> UpdateForecastAsync(int id, Forecast forecast)
    {
        var response = await _httpClient.PutAsJsonAsync($"forecasts/{id}", forecast);
        return response.IsSuccessStatusCode;
    }

    // DELETE request
    public async Task<bool> DeleteReportAsync(int id)
    {
        var response = await _httpClient.DeleteAsync($"reports/{id}");
        return response.IsSuccessStatusCode;
    }

    // GET with custom headers
    public async Task<string> GetAuthenticatedDataAsync(string token)
    {
        using var request = new HttpRequestMessage(HttpMethod.Get, "secure-data");
        request.Headers.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        var response = await _httpClient.SendAsync(request);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadAsStringAsync();
    }
}
```

**Register in Program.cs**:

```csharp
builder.Services.AddHttpClient<WeatherService>();
```

**Why TAP here**: HTTP calls are I/O-bound network operations. Using `await` prevents thread blocking while waiting for responses.

***

## 4. Parallel Async Operations with Task.WhenAll

### Use Case: Fetching Data from Multiple Sources Concurrently

When you need data from multiple independent sources, run them in parallel.

```csharp
public class DashboardService
{
    private readonly ProductService _productService;
    private readonly OrderService _orderService;
    private readonly CustomerService _customerService;

    public DashboardService(
        ProductService productService,
        OrderService orderService,
        CustomerService customerService)
    {
        _productService = productService;
        _orderService = orderService;
        _customerService = customerService;
    }

    // ✅ CORRECT: Run all queries in parallel
    public async Task<DashboardData> GetDashboardDataAsync()
    {
        // Start all tasks simultaneously
        var productsTask = _productService.GetProductsAsync("Electronics");
        var ordersTask = _orderService.GetRecentOrdersAsync();
        var customersTask = _customerService.GetActiveCustomersAsync();

        // Wait for all to complete
        await Task.WhenAll(productsTask, ordersTask, customersTask);

        // Retrieve results
        return new DashboardData
        {
            Products = await productsTask,
            Orders = await ordersTask,
            Customers = await customersTask
        };
    }
}
```

**Controller usage**:

```csharp
[HttpGet("dashboard")]
public async Task<ActionResult<DashboardData>> GetDashboard()
{
    var data = await _dashboardService.GetDashboardDataAsync();
    return Ok(data);
}
```

**Performance benefit**: Instead of 3 sequential waits (e.g., 300ms + 400ms + 200ms = 900ms), you wait for the longest one (~400ms).

***

## 5. Async Streams for Large Data Sets

### Use Case: Streaming Results from Database

For large result sets, use `IAsyncEnumerable<T>` to stream data as it becomes available.

```csharp
public class ReportService
{
    private readonly AppDbContext _context;

    public ReportService(AppDbContext context)
    {
        _context = context;
    }

    // Stream large dataset
    public async IAsyncEnumerable<Order> StreamOrdersAsync(
        DateTime startDate,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        await foreach (var order in _context.Orders
            .Where(o => o.OrderDate >= startDate)
            .AsAsyncEnumerable()
            .WithCancellation(cancellationToken))
        {
            yield return order;
        }
    }
}
```

**Controller with streaming response**:

```csharp
[HttpGet("orders/stream")]
public IAsyncEnumerable<Order> StreamOrders(
    [FromQuery] DateTime startDate,
    CancellationToken cancellationToken)
{
    return _reportService.StreamOrdersAsync(startDate, cancellationToken);
}
```

**Why async streams**: Avoids loading millions of records into memory at once.

***

## 6. Background Tasks with Hosted Services

### Use Case: Long-Running Background Processing

For background work in ASP.NET Core 8, use `BackgroundService` with TAP.

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(
        IServiceProvider serviceProvider,
        ILogger<OrderProcessingService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order Processing Service started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var orderService = scope.ServiceProvider
                    .GetRequiredService<OrderService>();

                await orderService.ProcessPendingOrdersAsync();
                
                // Wait 30 seconds before next run
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing orders");
            }
        }

        _logger.LogInformation("Order Processing Service stopped");
    }
}
```

**Register in Program.cs**:

```csharp
builder.Services.AddHostedService<OrderProcessingService>();
```


***

## 7. Cancellation Support with CancellationToken

### Use Case: User-Cancellable Long Operations

TAP integrates `CancellationToken` for cooperative cancellation.

```csharp
public class ExportService
{
    private readonly AppDbContext _context;

    public ExportService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<byte[]> ExportLargeReportAsync(
        CancellationToken cancellationToken)
    {
        var data = await _context.Orders
            .Where(o => o.Status == "Completed")
            .ToListAsync(cancellationToken); // Pass token to EF Core

        using var ms = new MemoryStream();
        
        foreach (var order in data)
        {
            // Check for cancellation before processing each item
            cancellationToken.ThrowIfCancellationRequested();
            
            // Simulate processing
            await Task.Delay(100, cancellationToken);
            
            // Write to stream...
        }

        return ms.ToArray();
    }
}
```

**Controller with timeout**:

```csharp
[HttpGet("export")]
public async Task<IActionResult> ExportReport(CancellationToken cancellationToken)
{
    using var cts = CancellationTokenSource
        .CreateLinkedTokenSource(cancellationToken);
    cts.CancelAfter(TimeSpan.FromMinutes(5)); // 5-minute timeout

    try
    {
        var data = await _exportService.ExportLargeReportAsync(cts.Token);
        return File(data, "application/pdf", "report.pdf");
    }
    catch (OperationCanceledException)
    {
        return StatusCode(408, "Request timed out");
    }
}
```


***

## 8. Exception Handling in TAP

### Use Case: Graceful Error Handling

TAP exceptions propagate naturally through `await`.

```csharp
public class OrderService
{
    private readonly AppDbContext _context;
    private readonly ILogger<OrderService> _logger;

    public async Task<Order> PlaceOrderAsync(Order order)
    {
        try
        {
            await _context.Orders.AddAsync(order);
            await _context.SaveChangesAsync();
            
            // Call external payment API
            await ProcessPaymentAsync(order.PaymentInfo);
            
            return order;
        }
        catch (DbUpdateException ex)
        {
            _logger.LogError(ex, "Database error placing order");
            throw new InvalidOperationException("Failed to save order", ex);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Payment processing failed");
            throw new InvalidOperationException("Payment failed", ex);
        }
    }

    private async Task ProcessPaymentAsync(PaymentInfo info)
    {
        // Simulated external API call
        await Task.Delay(1000);
    }
}
```


***

## 9. Task Composition Patterns

### Use Case: Complex Workflows

TAP enables clean composition of async operations.

```csharp
public class CheckoutService
{
    public async Task<CheckoutResult> ProcessCheckoutAsync(Cart cart)
    {
        // Step 1: Validate inventory (async)
        var validationResult = await ValidateInventoryAsync(cart);
        if (!validationResult.IsValid)
            return CheckoutResult.Failed(validationResult.Errors);

        // Step 2: Calculate totals (sync - fast)
        var totals = CalculateTotals(cart);

        // Step 3: Process payment and create order in parallel
        var paymentTask = ProcessPaymentAsync(totals.GrandTotal);
        var orderTask = CreateOrderAsync(cart, totals);

        await Task.WhenAll(paymentTask, orderTask);

        var payment = await paymentTask;
        var order = await orderTask;

        // Step 4: Send confirmation email (fire-and-forget)
        _ = SendConfirmationEmailAsync(order.Id);

        return CheckoutResult.Success(order, payment);
    }

    private async Task<ValidationResult> ValidateInventoryAsync(Cart cart)
    {
        await Task.Delay(100); // Simulated DB check
        return new ValidationResult { IsValid = true };
    }

    private Totals CalculateTotals(Cart cart)
    {
        // Fast, synchronous calculation
        return new Totals { GrandTotal = cart.Items.Sum(i => i.Price) };
    }

    private async Task<Payment> ProcessPaymentAsync(decimal amount)
    {
        await Task.Delay(200); // Simulated payment gateway
        return new Payment { Amount = amount, Status = "Completed" };
    }

    private async Task<Order> CreateOrderAsync(Cart cart, Totals totals)
    {
        await Task.Delay(150); // Simulated DB save
        return new Order { Id = 123, Total = totals.GrandTotal };
    }

    private async Task SendConfirmationEmailAsync(int orderId)
    {
        await Task.Delay(500); // Simulated email service
    }
}
```


***

## TAP Best Practices Summary

1. **I/O-bound operations**: Always use TAP for database, HTTP, file I/O
2. **Controller methods**: Return `Task<IActionResult>` or `Task<ActionResult<T>>`
3. **Avoid `async void`**: Only use in event handlers
4. **Pass CancellationToken**: Enable cancellation for long operations
5. **Use Task.WhenAll**: For parallel independent operations
6. **Don't block on async**: Never use `.Result` or `.Wait()` in async code
7. **Configure HttpClient properly**: Use `IHttpClientFactory`

