# .NET Examples

This page contains practical examples of using Mmate in .NET applications.

## Basic Examples

### Hello World Publisher

```csharp
using Mmate.Contracts;
using Mmate.Messaging;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Define a simple event
public class HelloEvent : Event
{
    public string Message { get; set; }
    public string From { get; set; }
    
    public HelloEvent() : base("HelloEvent") { }
}

// Program.cs
var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices((context, services) =>
{
    services.AddMmate(options =>
    {
        options.ConnectionString = "amqp://localhost";
    });
});

var host = builder.Build();

// Publish event
var publisher = host.Services.GetRequiredService<IMessagePublisher>();

await publisher.PublishEventAsync(new HelloEvent
{
    Message = "Hello, Mmate!",
    From = "example-app"
});

Console.WriteLine("Event published successfully");
```

### Hello World Consumer

```csharp
using Mmate.Contracts;
using Mmate.Messaging;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

// Handler
public class HelloHandler : IMessageHandler<HelloEvent>
{
    private readonly ILogger<HelloHandler> _logger;
    
    public HelloHandler(ILogger<HelloHandler> logger)
    {
        _logger = logger;
    }
    
    public async Task HandleAsync(HelloEvent message, MessageContext context)
    {
        _logger.LogInformation("Received: {Message} from {From}", 
            message.Message, message.From);
        await Task.CompletedTask;
    }
}

// Program.cs
var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices((context, services) =>
{
    services.AddMmate(options =>
    {
        options.ConnectionString = "amqp://localhost";
    });
    
    services.AddScoped<HelloHandler>();
    
    services.AddHostedService<MessageConsumerService>();
});

// Background service
public class MessageConsumerService : BackgroundService
{
    private readonly IMessageSubscriber _subscriber;
    private readonly HelloHandler _handler;
    
    public MessageConsumerService(
        IMessageSubscriber subscriber,
        HelloHandler handler)
    {
        _subscriber = subscriber;
        _handler = handler;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _subscriber.SubscribeAsync<HelloEvent>(
            "hello.queue", 
            _handler.HandleAsync,
            cancellationToken: stoppingToken);
        
        // Keep service running
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
}

var host = builder.Build();
await host.RunAsync();
```

## Request/Reply Pattern

### Service Implementation

```csharp
// Query definition
public class GetProductQuery : Query
{
    public string ProductId { get; set; }
    
    public GetProductQuery() : base("GetProductQuery") { }
}

// Reply definition
public class ProductReply : Reply
{
    public Product Product { get; set; }
    public string Error { get; set; }
    
    public static ProductReply Success(Product product)
    {
        return new ProductReply 
        { 
            Success = true, 
            Product = product 
        };
    }
    
    public static ProductReply Failure(string error)
    {
        return new ProductReply 
        { 
            Success = false, 
            Error = error 
        };
    }
}

public class Product
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public bool InStock { get; set; }
}

// Handler
public class ProductHandler : IRequestHandler<GetProductQuery, ProductReply>
{
    private readonly IProductCatalog _catalog;
    private readonly ILogger<ProductHandler> _logger;
    
    public ProductHandler(
        IProductCatalog catalog,
        ILogger<ProductHandler> logger)
    {
        _catalog = catalog;
        _logger = logger;
    }
    
    public async Task<ProductReply> HandleAsync(
        GetProductQuery query,
        RequestContext context)
    {
        try
        {
            var product = await _catalog.GetProductAsync(query.ProductId);
            if (product == null)
            {
                return ProductReply.Failure($"Product {query.ProductId} not found");
            }
            
            return ProductReply.Success(product);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving product {ProductId}", 
                query.ProductId);
            return ProductReply.Failure("An error occurred");
        }
    }
}

// Client using bridge
public class ProductService
{
    private readonly ISyncAsyncBridge _bridge;
    
    public ProductService(ISyncAsyncBridge bridge)
    {
        _bridge = bridge;
    }
    
    public async Task<Product> GetProductAsync(string productId)
    {
        var query = new GetProductQuery { ProductId = productId };
        
        var reply = await _bridge.SendAndWaitAsync<ProductReply>(
            query,
            timeout: TimeSpan.FromSeconds(10));
        
        if (!reply.Success)
        {
            throw new ProductNotFoundException(reply.Error);
        }
        
        return reply.Product;
    }
}
```

## Command Handler with Validation

```csharp
public class CreateOrderCommand : Command
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public ShippingAddress ShippingAddress { get; set; }
    
    public CreateOrderCommand() : base("CreateOrderCommand")
    {
        Items = new List<OrderItem>();
    }
    
    public override ValidationResult Validate()
    {
        var errors = new List<string>();
        
        if (string.IsNullOrEmpty(CustomerId))
            errors.Add("Customer ID is required");
        
        if (Items?.Any() != true)
            errors.Add("Order must have at least one item");
        
        foreach (var (item, index) in Items.Select((item, i) => (item, i)))
        {
            if (string.IsNullOrEmpty(item.ProductId))
                errors.Add($"Item {index}: Product ID is required");
            
            if (item.Quantity <= 0)
                errors.Add($"Item {index}: Quantity must be positive");
            
            if (item.Price < 0)
                errors.Add($"Item {index}: Price cannot be negative");
        }
        
        var addressValidation = ShippingAddress?.Validate();
        if (addressValidation?.IsValid == false)
            errors.AddRange(addressValidation.Errors);
        
        return errors.Any() 
            ? ValidationResult.Failure(errors) 
            : ValidationResult.Success();
    }
}

public class OrderItem
{
    public string ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    private readonly IOrderService _orderService;
    private readonly IMessagePublisher _publisher;
    private readonly ILogger<OrderHandler> _logger;
    
    public OrderHandler(
        IOrderService orderService,
        IMessagePublisher publisher,
        ILogger<OrderHandler> logger)
    {
        _orderService = orderService;
        _publisher = publisher;
        _logger = logger;
    }
    
    public async Task HandleAsync(
        CreateOrderCommand command,
        MessageContext context)
    {
        // Validation is done by interceptor, but we can double-check
        var validation = command.Validate();
        if (!validation.IsValid)
        {
            throw new ValidationException(validation.Errors);
        }
        
        // Create order
        var orderId = await _orderService.CreateOrderAsync(command);
        
        // Publish event
        var @event = new OrderCreatedEvent
        {
            OrderId = orderId,
            CustomerId = command.CustomerId,
            TotalAmount = command.Items.Sum(i => i.Quantity * i.Price),
            CreatedAt = DateTime.UtcNow
        };
        
        await _publisher.PublishEventAsync(@event);
        
        _logger.LogInformation("Order {OrderId} created successfully", orderId);
    }
}
```

## StageFlow Workflow

```csharp
// Workflow context
public class OrderFulfillmentContext : WorkflowContext
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    
    // Stage results
    public string PaymentId { get; set; }
    public bool InventoryReserved { get; set; }
    public string ShipmentId { get; set; }
    public string TrackingNumber { get; set; }
}

// Stage implementations
public class ValidateOrderStage : IWorkflowStage<OrderFulfillmentContext>
{
    public async Task ExecuteAsync(
        OrderFulfillmentContext context,
        CancellationToken cancellationToken)
    {
        if (string.IsNullOrEmpty(context.OrderId))
            throw new ValidationException("Order ID is required");
        
        if (!context.Items?.Any() ?? true)
            throw new ValidationException("Order has no items");
        
        await Task.CompletedTask;
    }
}

public class ProcessPaymentStage : IWorkflowStage<OrderFulfillmentContext>
{
    private readonly IPaymentService _paymentService;
    
    public ProcessPaymentStage(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
    
    public async Task ExecuteAsync(
        OrderFulfillmentContext context,
        CancellationToken cancellationToken)
    {
        var request = new PaymentRequest
        {
            OrderId = context.OrderId,
            CustomerId = context.CustomerId,
            Amount = context.Items.Sum(i => i.Quantity * i.Price)
        };
        
        var result = await _paymentService.ProcessPaymentAsync(request);
        
        if (!result.Success)
            throw new PaymentException(result.Error);
        
        context.PaymentId = result.PaymentId;
    }
}

public class RefundPaymentStage : ICompensationStage<OrderFulfillmentContext>
{
    private readonly IPaymentService _paymentService;
    
    public RefundPaymentStage(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
    
    public async Task CompensateAsync(
        OrderFulfillmentContext context,
        Exception failureReason,
        CancellationToken cancellationToken)
    {
        if (!string.IsNullOrEmpty(context.PaymentId))
        {
            await _paymentService.RefundPaymentAsync(
                context.PaymentId,
                $"Order processing failed: {failureReason.Message}");
        }
    }
}

// Workflow definition
public class OrderFulfillmentWorkflow : Workflow<OrderFulfillmentContext>
{
    protected override void Configure(IWorkflowBuilder<OrderFulfillmentContext> builder)
    {
        builder
            .AddStage<ValidateOrderStage>()
            
            .AddStage<ProcessPaymentStage>()
                .WithCompensation<RefundPaymentStage>()
            
            .AddStage<ReserveInventoryStage>()
                .WithCompensation<ReleaseInventoryStage>()
                .WithRetry(policy => policy
                    .MaxAttempts(3)
                    .WithExponentialBackoff())
            
            .AddStage<CreateShipmentStage>()
                .WithTimeout(TimeSpan.FromMinutes(5))
            
            .AddStage<SendNotificationStage>()
                .ContinueOnFailure(); // Non-critical stage
    }
}

// Usage
public class OrderService
{
    private readonly IWorkflowEngine _workflowEngine;
    
    public OrderService(IWorkflowEngine workflowEngine)
    {
        _workflowEngine = workflowEngine;
    }
    
    public async Task<WorkflowResult> ProcessOrderAsync(string orderId)
    {
        var context = new OrderFulfillmentContext
        {
            OrderId = orderId,
            // Load order data
        };
        
        var result = await _workflowEngine.ExecuteAsync(
            new OrderFulfillmentWorkflow(),
            context);
        
        if (!result.Success)
        {
            _logger.LogError("Order processing failed: {Error}", result.Error);
        }
        
        return result;
    }
}
```

## Interceptor Example

```csharp
// Custom metrics interceptor
public class MetricsInterceptor : IMessageInterceptor
{
    private readonly IMetrics _metrics;
    
    public MetricsInterceptor(IMetrics metrics)
    {
        _metrics = metrics;
    }
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        var messageType = context.Message.GetType().Name;
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            context = await next(context);
            
            _metrics.Counter("messages_processed", 1)
                .WithTag("type", messageType)
                .WithTag("status", "success")
                .Publish();
            
            return context;
        }
        catch (Exception ex)
        {
            _metrics.Counter("messages_processed", 1)
                .WithTag("type", messageType)
                .WithTag("status", "error")
                .WithTag("error_type", ex.GetType().Name)
                .Publish();
            
            throw;
        }
        finally
        {
            _metrics.Histogram("message_processing_duration", stopwatch.Elapsed.TotalSeconds)
                .WithTag("type", messageType)
                .Publish();
        }
    }
}

// Registration
services.AddMmate(options =>
{
    options.AddInterceptor<MetricsInterceptor>();
    options.AddInterceptor<LoggingInterceptor>();
    options.AddInterceptor<TracingInterceptor>();
    options.AddInterceptor<ValidationInterceptor>();
});
```

## Consumer Group Example

```csharp
// Configure consumer group
services.AddMmate(options =>
{
    options.ConsumerGroups.Add("order-processors", group =>
    {
        group.ConsumerCount = 5;
        group.PrefetchCount = 2;
        group.Queue = "cmd.orders.process";
    });
});

// Scaled handler
[ConsumerGroup("order-processors")]
public class OrderProcessor : IMessageHandler<ProcessOrderCommand>
{
    private readonly ILogger<OrderProcessor> _logger;
    private readonly int _workerId;
    
    public OrderProcessor(ILogger<OrderProcessor> logger)
    {
        _logger = logger;
        _workerId = Thread.CurrentThread.ManagedThreadId;
    }
    
    public async Task HandleAsync(
        ProcessOrderCommand command,
        MessageContext context)
    {
        _logger.LogInformation("Worker {WorkerId} processing order {OrderId}",
            _workerId, command.OrderId);
        
        // Process order
        await ProcessOrderAsync(command);
    }
}

// Or manually create multiple instances
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly List<Task> _workers = new();
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        for (int i = 0; i < 5; i++)
        {
            var workerId = i;
            _workers.Add(Task.Run(async () =>
            {
                using var scope = _serviceProvider.CreateScope();
                var subscriber = scope.ServiceProvider
                    .GetRequiredService<IMessageSubscriber>();
                var handler = scope.ServiceProvider
                    .GetRequiredService<OrderProcessor>();
                
                await subscriber.SubscribeAsync<ProcessOrderCommand>(
                    $"order-processor-{workerId}",
                    handler.HandleAsync,
                    options => options.WithPrefetchCount(2),
                    cancellationToken: stoppingToken);
            }));
        }
        
        await Task.WhenAll(_workers);
    }
}
```

## Error Handling Example

```csharp
// Custom error types
public class ValidationException : Exception
{
    public List<ValidationError> Errors { get; }
    
    public ValidationException(params string[] errors)
        : base("Validation failed")
    {
        Errors = errors.Select(e => new ValidationError { Message = e }).ToList();
    }
}

public class TransientException : Exception
{
    public TransientException(string message, Exception inner = null)
        : base(message, inner) { }
}

// Handler with error handling
public class PaymentHandler : IMessageHandler<ProcessPaymentCommand>
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly ILogger<PaymentHandler> _logger;
    
    public async Task HandleAsync(
        ProcessPaymentCommand command,
        MessageContext context)
    {
        // Validation errors - don't retry
        if (command.Amount <= 0)
        {
            throw new ValidationException("Amount must be positive");
        }
        
        try
        {
            var result = await _paymentGateway.ProcessAsync(command);
            
            if (!result.Success)
            {
                // Business logic failure - don't retry
                throw new PaymentDeclinedException(result.DeclineReason);
            }
            
            // Publish success event
            await _publisher.PublishEventAsync(new PaymentProcessedEvent
            {
                PaymentId = result.PaymentId,
                OrderId = command.OrderId,
                Amount = command.Amount,
                ProcessedAt = DateTime.UtcNow
            });
        }
        catch (NetworkException ex)
        {
            // Network errors - retry
            _logger.LogWarning(ex, "Network error, will retry");
            throw new TransientException("Network error", ex);
        }
        catch (RateLimitException ex)
        {
            // Rate limit - retry with backoff
            context.Headers["retry-after"] = ex.RetryAfter.ToString();
            throw new TransientException($"Rate limited, retry after {ex.RetryAfter}", ex);
        }
        catch (Exception ex)
        {
            // Unknown error - don't retry
            _logger.LogError(ex, "Payment processing failed");
            throw new PermanentException("Payment processing failed", ex);
        }
    }
}
```

## Health Check Example

```csharp
// Custom health check
public class MessagingHealthCheck : IHealthCheck
{
    private readonly IMessagePublisher _publisher;
    private readonly IConnectionManager _connectionManager;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Check connection
            if (!_connectionManager.IsConnected)
            {
                return HealthCheckResult.Unhealthy(
                    "RabbitMQ connection is not established");
            }
            
            // Try to publish a test message
            await _publisher.PublishEventAsync(
                new HealthCheckEvent { Timestamp = DateTime.UtcNow },
                cancellationToken);
            
            return HealthCheckResult.Healthy("Messaging system is operational");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Messaging system check failed",
                exception: ex);
        }
    }
}

// Registration
builder.Services.AddHealthChecks()
    .AddRabbitMQ(rabbitConnectionString: "amqp://localhost",
        name: "rabbitmq",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "messaging", "infrastructure" })
    .AddCheck<MessagingHealthCheck>("messaging",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "messaging", "custom" });

// Map endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("infrastructure")
});
```

## Testing Example

```csharp
using Xunit;
using Moq;
using Microsoft.Extensions.Logging;

public class OrderHandlerTests
{
    [Fact]
    public async Task HandleAsync_ValidCommand_CreatesOrderAndPublishesEvent()
    {
        // Arrange
        var mockOrderService = new Mock<IOrderService>();
        var mockPublisher = new Mock<IMessagePublisher>();
        var mockLogger = new Mock<ILogger<OrderHandler>>();
        
        mockOrderService
            .Setup(x => x.CreateOrderAsync(It.IsAny<CreateOrderCommand>()))
            .ReturnsAsync("ORDER-123");
        
        var handler = new OrderHandler(
            mockOrderService.Object,
            mockPublisher.Object,
            mockLogger.Object);
        
        var command = new CreateOrderCommand
        {
            CustomerId = "CUST-123",
            Items = new List<OrderItem>
            {
                new() { ProductId = "PROD-1", Quantity = 2, Price = 10.00m }
            }
        };
        
        // Act
        await handler.HandleAsync(command, new MessageContext());
        
        // Assert
        mockOrderService.Verify(
            x => x.CreateOrderAsync(It.Is<CreateOrderCommand>(
                c => c.CustomerId == "CUST-123")),
            Times.Once);
        
        mockPublisher.Verify(
            x => x.PublishEventAsync(It.Is<OrderCreatedEvent>(
                e => e.OrderId == "ORDER-123" && e.TotalAmount == 20.00m)),
            Times.Once);
    }
    
    [Fact]
    public async Task HandleAsync_InvalidCommand_ThrowsValidationException()
    {
        // Arrange
        var handler = new OrderHandler(
            Mock.Of<IOrderService>(),
            Mock.Of<IMessagePublisher>(),
            Mock.Of<ILogger<OrderHandler>>());
        
        var command = new CreateOrderCommand
        {
            CustomerId = "", // Invalid
            Items = new List<OrderItem>()
        };
        
        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(
            () => handler.HandleAsync(command, new MessageContext()));
    }
}

// Integration test
public class OrderWorkflowIntegrationTests : IClassFixture<TestFixture>
{
    private readonly TestFixture _fixture;
    
    public OrderWorkflowIntegrationTests(TestFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public async Task OrderWorkflow_CompleteFlow_Success()
    {
        // Arrange
        var services = new ServiceCollection();
        services.AddMmate(options =>
        {
            options.ConnectionString = _fixture.RabbitMqConnectionString;
        });
        services.AddSingleton<IPaymentService, MockPaymentService>();
        services.AddSingleton<IInventoryService, MockInventoryService>();
        
        var provider = services.BuildServiceProvider();
        var workflowEngine = provider.GetRequiredService<IWorkflowEngine>();
        
        var context = new OrderFulfillmentContext
        {
            OrderId = "TEST-ORDER-123",
            CustomerId = "TEST-CUST-456",
            Items = new List<OrderItem>
            {
                new() { ProductId = "TEST-1", Quantity = 1, Price = 50.00m }
            }
        };
        
        // Act
        var result = await workflowEngine.ExecuteAsync(
            new OrderFulfillmentWorkflow(),
            context);
        
        // Assert
        Assert.True(result.Success);
        Assert.NotNull(context.PaymentId);
        Assert.True(context.InventoryReserved);
        Assert.NotNull(context.ShipmentId);
    }
}
```

## ASP.NET Core Integration

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add Mmate
builder.Services.AddMmate(builder.Configuration);

// Add handlers
builder.Services.AddMmateHandlers(typeof(Program).Assembly);

// Add API controllers
builder.Services.AddControllers();

var app = builder.Build();

// Use Mmate middleware
app.UseMmate();

// Map health checks
app.MapHealthChecks("/health");

// API endpoint that publishes messages
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    IMessagePublisher publisher,
    ISyncAsyncBridge bridge) =>
{
    // Validate
    if (!request.IsValid)
        return Results.BadRequest(request.Errors);
    
    // Send command and wait for result
    var command = new CreateOrderCommand
    {
        CustomerId = request.CustomerId,
        Items = request.Items
    };
    
    try
    {
        var reply = await bridge.SendAndWaitAsync<OrderReply>(
            command,
            timeout: TimeSpan.FromSeconds(30));
        
        if (reply.Success)
        {
            return Results.Ok(new { OrderId = reply.OrderId });
        }
        
        return Results.BadRequest(new { Error = reply.Error });
    }
    catch (TimeoutException)
    {
        return Results.StatusCode(504); // Gateway Timeout
    }
});

app.Run();
```