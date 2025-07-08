# .NET Platform Documentation

> **🟢 .NET Implementation Status**: The .NET implementation provides enterprise-ready messaging capabilities with 85% feature parity to the Go implementation. It includes modern middleware architecture, batch publishing, health monitoring, StageFlow workflows, and full ASP.NET Core integration.

This section contains .NET-specific documentation for the comprehensive messaging framework available in Mmate .NET.

## Contents

- [API Reference](api-reference.md) - Complete .NET API documentation
- [Configuration](configuration.md) - .NET-specific configuration options
- [Examples](examples.md) - .NET code examples and patterns

## Quick Links

- [Getting Started Guide](../../getting-started/dotnet.md)
- [.NET Repository](https://github.com/mmate/mmate-dotnet)

## .NET-Specific Features

### Modern Middleware Architecture
Built on ASP.NET Core-inspired middleware pipeline:

```csharp
services.AddMmateMessaging(options =>
{
    options.RabbitMqConnection = "amqp://localhost";
})
.WithMiddleware(pipeline =>
{
    pipeline.UseLogging();
    pipeline.UseMetrics();
    pipeline.UseRetryPolicy();
    pipeline.UseCircuitBreaker();
});
```

### Batch Publishing Support
High-performance batch operations:

```csharp
var batchPublisher = serviceProvider.GetRequiredService<IBatchPublisher>();
await batchPublisher.PublishBatchAsync(messages, 
    options: new BatchPublishingOptions
    {
        MaxBatchSize = 100,
        FlushIntervalMs = 1000
    });
```

### Advanced Health Monitoring
Comprehensive health checks and metrics:

```csharp
services.AddHealthChecks()
    .AddMmateHealthChecks()
    .AddRabbitMqConnectionCheck()
    .AddCircuitBreakerCheck();

services.AddMmateMetrics(options =>
{
    options.EnableDetailedMetrics = true;
    options.MetricsPrefix = "myapp.messaging";
});
```

### Native ASP.NET Core Integration
Seamless integration with the ASP.NET Core ecosystem:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMmateMessaging(builder.Configuration);
builder.Services.AddMmateHandlers(); // Auto-discovery

var app = builder.Build();
app.UseMmateMiddleware();
app.MapMmateHealthChecks("/health/messaging");
```

### Enterprise StageFlow Workflows
Multi-stage workflow orchestration with persistence:

```csharp
services.AddStageFlow(options =>
{
    options.UseInMemoryState(); // or UseRedisState() for production
    options.MaxStageConcurrency = 10;
    options.DefaultStageTimeout = TimeSpan.FromMinutes(5);
});

// Define workflows with compensation patterns
var workflow = flowFactory.CreateFlow<OrderRequest, OrderState>("OrderProcessing");
workflow.Stage<OrderRequest>(ValidateOrderAsync)
        .Stage<ValidateOrderResponse>(ProcessPaymentAsync)
        .Stage<ProcessPaymentResponse>(CreateShipmentAsync)
        .LastStage<CreateShipmentResponse, OrderResult>(CompleteOrderAsync);
```

### Consumer Groups and Scaling
Automatic consumer group management:

```csharp
services.AddMmateMessaging()
    .WithConsumerGroups(groups =>
    {
        groups.AddGroup("order-processing", size: 5)
              .AddGroup("payment-processing", size: 3)
              .AddGroup("notification-sending", size: 10);
    });
```

## Common Patterns

### Configuration
appsettings.json with strongly-typed options:

```json
{
  "Mmate": {
    "ConnectionString": "amqp://localhost",
    "PrefetchCount": 10,
    "RetryPolicy": {
      "MaxAttempts": 3,
      "InitialDelay": "00:00:01"
    }
  }
}
```

```csharp
services.Configure<MmateOptions>(
    Configuration.GetSection("Mmate"));
```

### Testing
xUnit with Moq for mocking:

```csharp
[Fact]
public async Task HandleAsync_ValidCommand_Success()
{
    // Arrange
    var mockService = new Mock<IOrderService>();
    var handler = new OrderHandler(mockService.Object);
    
    // Act
    await handler.HandleAsync(command, context);
    
    // Assert
    mockService.Verify(x => x.CreateOrder(It.IsAny<Order>()), Times.Once);
}
```

### Logging
Structured logging with ILogger:

```csharp
_logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
    order.Id, order.CustomerId);
```

## Integration Features

### ASP.NET Core
Full integration with ASP.NET Core:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMmate(builder.Configuration);
builder.Services.AddControllers();
builder.Services.AddHealthChecks().AddRabbitMQ();

var app = builder.Build();

app.UseMmate();
app.MapHealthChecks("/health");
app.MapControllers();

app.Run();
```

### Entity Framework Core
Works seamlessly with EF Core:

```csharp
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    private readonly OrderDbContext _dbContext;
    
    public async Task HandleAsync(CreateOrderCommand command, MessageContext context)
    {
        var order = new Order { /* ... */ };
        _dbContext.Orders.Add(order);
        await _dbContext.SaveChangesAsync();
    }
}
```

### Background Services
Implement as hosted services:

```csharp
public class MessageConsumerService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _subscriber.SubscribeAsync<OrderEvent>(
            "order-events",
            _handler.HandleAsync,
            cancellationToken: stoppingToken);
    }
}
```

## Best Practices

1. **Use dependency injection** - Leverage the built-in DI container
2. **Configure via appsettings** - Use configuration providers
3. **Implement health checks** - Monitor service health
4. **Use structured logging** - Include context in logs
5. **Handle graceful shutdown** - Implement IHostedService properly