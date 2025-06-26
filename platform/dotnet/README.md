# .NET Platform Documentation

This section contains .NET-specific documentation for Mmate.

## Contents

- [API Reference](api-reference.md) - Complete .NET API documentation
- [Configuration](configuration.md) - .NET-specific configuration options
- [Examples](examples.md) - .NET code examples and patterns

## Quick Links

- [Getting Started Guide](../../getting-started/dotnet.md)
- [.NET Repository](https://github.com/mmate/mmate-dotnet)

## .NET-Specific Features

### Built-in Dependency Injection
Integrated with ASP.NET Core DI container:

```csharp
services.AddMmate(options =>
{
    options.ConnectionString = "amqp://localhost";
});

services.AddScoped<IOrderService, OrderService>();
services.AddMmateHandlers(typeof(Program).Assembly);
```

### Async/Await Pattern
All operations support async/await:

```csharp
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    command,
    timeout: TimeSpan.FromSeconds(30));
```

### Exception-Based Error Handling
Uses exceptions for error handling:

```csharp
try
{
    await ProcessOrderAsync(order);
}
catch (ValidationException ex)
{
    // Handle validation errors
}
catch (TransientException ex)
{
    // Retry transient errors
}
```

### Attribute-Based Configuration
Configure behavior with attributes:

```csharp
[MessageHandler(Queue = "orders", PrefetchCount = 10)]
[RetryPolicy(MaxAttempts = 3)]
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    // Implementation
}
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