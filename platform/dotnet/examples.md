# .NET Examples

> **🟢 .NET Enterprise Features**: The .NET implementation provides comprehensive enterprise messaging capabilities with modern middleware architecture, batch publishing, advanced monitoring, and StageFlow workflows.

This page contains practical examples of using the full enterprise messaging features available in the .NET implementation.

## Modern Middleware Architecture

### Comprehensive Middleware Setup

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Mmate with full middleware pipeline
builder.Services.AddMmateMessaging(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("RabbitMQ");
    options.ExchangeName = "myapp.exchange";
    options.QueuePrefix = "myapp.";
})
.WithMiddleware(pipeline =>
{
    // Request/response logging with detailed context
    pipeline.UseLogging(opts =>
    {
        opts.LogLevel = LogLevel.Information;
        opts.LogMessageContent = true;
        opts.LogHeaders = true;
        opts.MessageTemplate = "Processing {MessageType} [ID: {MessageId}, Correlation: {CorrelationId}]";
    });
    
    // Detailed metrics collection
    pipeline.UseMetrics(opts =>
    {
        opts.EnableDetailedMetrics = true;
        opts.RecordProcessingTime = true;
        opts.RecordMessageSize = true;
        opts.MetricsPrefix = "myapp.messaging";
    });
    
    // Circuit breaker for reliability
    pipeline.UseCircuitBreaker(opts =>
    {
        opts.FailureThreshold = 5;
        opts.OpenTimeout = TimeSpan.FromSeconds(30);
        opts.HalfOpenTimeout = TimeSpan.FromSeconds(10);
        opts.CountAsFailure = ex => !(ex is ValidationException);
    });
    
    // Retry policy with exponential backoff
    pipeline.UseRetryPolicy(opts =>
    {
        opts.MaxAttempts = 3;
        opts.InitialDelay = TimeSpan.FromSeconds(1);
        opts.MaxDelay = TimeSpan.FromMinutes(1);
        opts.BackoffMultiplier = 2.0;
        opts.ShouldRetry = ex => ex is TransientException || ex is TimeoutException;
    });
    
    // Centralized error handling
    pipeline.UseErrorHandling(opts =>
    {
        opts.LogErrors = true;
        opts.SendToDeadLetter = true;
        opts.ErrorDetailsInHeaders = true;
    });
});

var app = builder.Build();
app.UseMmateMiddleware();
app.Run();
```

### Custom Middleware Example

```csharp
public class AuthenticationMiddleware : IMessageMiddleware
{
    private readonly ILogger<AuthenticationMiddleware> _logger;
    private readonly IAuthenticationService _authService;

    public AuthenticationMiddleware(
        ILogger<AuthenticationMiddleware> logger,
        IAuthenticationService authService)
    {
        _logger = logger;
        _authService = authService;
    }

    public async Task InvokeAsync(MessageContext context, MessageDelegate next)
    {
        // Extract auth token from headers
        var token = context.GetHeader<string>("Authorization");
        
        if (string.IsNullOrEmpty(token))
        {
            _logger.LogWarning("Missing authorization header for message {MessageId}", 
                context.GetHeader<string>("MessageId"));
            throw new UnauthorizedException("Authorization required");
        }

        // Validate token
        var principal = await _authService.ValidateTokenAsync(token);
        if (principal == null)
        {
            _logger.LogWarning("Invalid authorization token for message {MessageId}", 
                context.GetHeader<string>("MessageId"));
            throw new UnauthorizedException("Invalid token");
        }

        // Add user context
        context.AddProperty("User", principal);
        context.AddProperty("UserId", principal.Identity.Name);

        _logger.LogDebug("Authenticated user {UserId} for message {MessageId}", 
            principal.Identity.Name, context.GetHeader<string>("MessageId"));

        await next(context);
    }
}

// Registration
services.AddMmateMessaging()
    .WithMiddleware(pipeline =>
    {
        pipeline.Use<AuthenticationMiddleware>();
        pipeline.UseLogging();
        // ... other middleware
    });
```

## Batch Publishing Examples

### High-Volume Event Publishing

```csharp
public class EventPublishingService : IHostedService
{
    private readonly IBatchPublisher _batchPublisher;
    private readonly ILogger<EventPublishingService> _logger;
    private readonly Timer _flushTimer;

    public EventPublishingService(
        IBatchPublisher batchPublisher, 
        ILogger<EventPublishingService> logger)
    {
        _batchPublisher = batchPublisher;
        _logger = logger;
    }

    public async Task PublishOrderEventsAsync(IEnumerable<OrderEvent> events)
    {
        var options = new BatchPublishingOptions
        {
            MaxBatchSize = 100,
            FlushInterval = TimeSpan.FromSeconds(5),
            WaitForConfirmation = true,
            ConfirmationTimeout = TimeSpan.FromSeconds(30)
        };

        var result = await _batchPublisher.PublishBatchWithResultAsync(events, options);

        if (!result.Success)
        {
            _logger.LogError("Batch publishing failed. {FailedCount}/{TotalCount} messages failed",
                result.FailedMessages, result.TotalMessages);
                
            foreach (var error in result.Errors)
            {
                _logger.LogError("Message at index {Index} failed: {Error}",
                    error.MessageIndex, error.ErrorMessage);
            }
        }
        else
        {
            _logger.LogInformation("Successfully published {Count} events in {Duration}ms",
                result.SuccessfulMessages, result.Duration.TotalMilliseconds);
        }
    }
}
```

### Smart Batching with Auto-Flush

```csharp
public class SmartOrderProcessor
{
    private readonly IBatchPublisher _batchPublisher;
    private readonly IBatch<OrderProcessedEvent> _eventBatch;

    public SmartOrderProcessor(IBatchPublisher batchPublisher)
    {
        _batchPublisher = batchPublisher;
        _eventBatch = _batchPublisher.CreateBatch<OrderProcessedEvent>(new BatchPublishingOptions
        {
            MaxBatchSize = 50,
            FlushInterval = TimeSpan.FromSeconds(2),
            EnableAutoFlush = true
        });
    }

    public async Task ProcessOrderAsync(Order order)
    {
        // Process the order
        await ProcessOrderLogicAsync(order);

        // Add event to batch (will auto-flush when full or after interval)
        _eventBatch.Add(new OrderProcessedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            ProcessedAt = DateTime.UtcNow,
            Status = "Completed"
        });

        // Manually flush if priority event
        if (order.Priority == OrderPriority.Urgent)
        {
            await _eventBatch.PublishAsync();
        }
    }
}
```

## Advanced Health Monitoring

### Comprehensive Health Checks

```csharp
public class MessagingHealthService : BackgroundService
{
    private readonly IAdvancedMetricsCollector _metrics;
    private readonly IBrokerMonitor _brokerMonitor;
    private readonly ILogger<MessagingHealthService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CollectHealthMetricsAsync();
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }

    private async Task CollectHealthMetricsAsync()
    {
        try
        {
            // Get detailed metrics
            var summary = await _metrics.GetMetricsSummaryAsync(TimeSpan.FromMinutes(5));
            
            // Monitor queue depths
            var queues = await _brokerMonitor.GetQueuesAsync(includeEmpty: false);
            
            foreach (var queue in queues)
            {
                _metrics.RecordQueueDepth(queue.Name, queue.MessageCount);
                _metrics.RecordConsumerCount(queue.Name, queue.ConsumerCount);
                
                // Alert on high queue depth
                if (queue.MessageCount > 1000)
                {
                    _logger.LogWarning("High queue depth detected: {QueueName} has {MessageCount} messages",
                        queue.Name, queue.MessageCount);
                }
            }

            // Check error rates
            var errorAnalysis = await _metrics.GetErrorAnalysisAsync(TimeSpan.FromMinutes(15));
            if (errorAnalysis.OverallErrorRate > 0.05) // 5% error rate threshold
            {
                _logger.LogWarning("High error rate detected: {ErrorRate:P2} in the last 15 minutes",
                    errorAnalysis.OverallErrorRate);
            }

            // Monitor latency
            var latencyStats = await _metrics.GetLatencyStatsAsync("message-processing", TimeSpan.FromMinutes(10));
            if (latencyStats.P95 > TimeSpan.FromSeconds(5))
            {
                _logger.LogWarning("High latency detected: P95 is {P95}ms",
                    latencyStats.P95.TotalMilliseconds);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to collect health metrics");
        }
    }
}

// Registration with health checks
builder.Services.AddHealthChecks()
    .AddMmateHealthChecks()
    .AddCheck<MessagingHealthService>("messaging-detailed")
    .AddCheck("circuit-breaker", () =>
    {
        var circuitBreaker = serviceProvider.GetService<ICircuitBreakerService>();
        return circuitBreaker.State == CircuitBreakerState.Open
            ? HealthCheckResult.Unhealthy("Circuit breaker is open")
            : HealthCheckResult.Healthy();
    });
```

## Enterprise StageFlow Workflows

### Complex Order Fulfillment Workflow

```csharp
public class OrderFulfillmentWorkflow
{
    private readonly IFlowEndpointFactory _factory;

    public void ConfigureWorkflow()
    {
        var workflow = _factory.CreateFlow<OrderCreatedEvent, OrderFulfillmentState>("OrderFulfillment");

        // Stage 1: Validate order and reserve inventory
        workflow.Stage<OrderCreatedEvent>(async (context, state, orderEvent) =>
        {
            state.OrderId = orderEvent.OrderId;
            state.CustomerId = orderEvent.CustomerId;
            state.Items = orderEvent.Items;
            state.Status = "ValidatingInventory";

            // Validate inventory with timeout and retry
            var inventoryRequest = new ValidateInventoryRequest
            {
                OrderId = state.OrderId,
                Items = state.Items
            };

            await context.RequestAsync("inventory.validate", inventoryRequest, 
                timeout: TimeSpan.FromSeconds(30));
        });

        // Stage 2: Process payment with compensation
        workflow.Stage<InventoryValidatedEvent>(async (context, state, inventoryEvent) =>
        {
            if (!inventoryEvent.IsValid)
            {
                state.Status = "Failed";
                state.FailureReason = "Insufficient inventory";
                
                // Publish failure event
                await context.PublishAsync(new OrderFulfillmentFailedEvent
                {
                    OrderId = state.OrderId,
                    Reason = state.FailureReason
                });
                return;
            }

            state.Status = "ProcessingPayment";
            state.InventoryReservationId = inventoryEvent.ReservationId;

            // Store compensation data for potential rollback
            await context.SetCompensationDataAsync("InventoryReservation", new
            {
                ReservationId = inventoryEvent.ReservationId,
                Items = state.Items
            });

            var paymentRequest = new ProcessPaymentRequest
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                Amount = inventoryEvent.TotalAmount
            };

            await context.RequestAsync("payment.process", paymentRequest,
                timeout: TimeSpan.FromSeconds(45));
        });

        // Stage 3: Create shipment
        workflow.Stage<PaymentProcessedEvent>(async (context, state, paymentEvent) =>
        {
            if (!paymentEvent.Success)
            {
                state.Status = "PaymentFailed";
                state.FailureReason = paymentEvent.FailureReason;

                // Trigger compensation - release inventory
                await context.CompensateAsync("InventoryReservation", async (compensationData) =>
                {
                    await context.RequestAsync("inventory.release", new ReleaseInventoryRequest
                    {
                        ReservationId = compensationData.ReservationId
                    });
                });

                await context.PublishAsync(new OrderFulfillmentFailedEvent
                {
                    OrderId = state.OrderId,
                    Reason = state.FailureReason
                });
                return;
            }

            state.Status = "CreatingShipment";
            state.PaymentId = paymentEvent.PaymentId;

            var shipmentRequest = new CreateShipmentRequest
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                Items = state.Items,
                ShippingAddress = paymentEvent.ShippingAddress
            };

            await context.RequestAsync("shipment.create", shipmentRequest);
        });

        // Final stage: Complete fulfillment
        workflow.LastStage<ShipmentCreatedEvent, OrderFulfillmentResult>(async (context, state, shipmentEvent) =>
        {
            state.Status = "Completed";
            state.ShipmentId = shipmentEvent.ShipmentId;
            state.CompletedAt = DateTime.UtcNow;

            // Publish completion event
            await context.PublishAsync(new OrderFulfilledEvent
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                PaymentId = state.PaymentId,
                ShipmentId = state.ShipmentId,
                CompletedAt = state.CompletedAt
            });

            return new OrderFulfillmentResult
            {
                Success = true,
                OrderId = state.OrderId,
                ShipmentId = state.ShipmentId,
                Status = state.Status
            };
        });

        // Configure error handling and compensation
        workflow.OnError(async (context, state, exception) =>
        {
            _logger.LogError(exception, "Order fulfillment failed for order {OrderId}", state.OrderId);
            
            // Trigger all compensation actions
            await context.CompensateAllAsync();
            
            // Publish error event
            await context.PublishAsync(new OrderFulfillmentFailedEvent
            {
                OrderId = state.OrderId,
                Reason = exception.Message
            });
        });
    }
}
```

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
    services.AddMmateMessaging(options =>
    {
        options.RabbitMqConnection = "amqp://localhost";
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
    services.AddMmateMessaging(options =>
    {
        options.RabbitMqConnection = "amqp://localhost";
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

// Note: This example uses the SyncAsyncBridge component which provides basic request/reply
// For advanced features like acknowledgment tracking, use the Go implementation
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

## Basic Pipeline Example

```csharp
// Simple pipeline context
public class OrderProcessingContext : IStageContext
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public Dictionary<string, object> Data { get; set; } = new();
}

// Basic stage implementation  
public class ValidateOrderStage : IStage
{
    public async Task ExecuteAsync(IStageContext context)
    {
        var orderContext = context as OrderProcessingContext;
        
        if (string.IsNullOrEmpty(orderContext?.OrderId))
            throw new ValidationException("Order ID is required");
        
        if (!orderContext.Items?.Any() ?? true)
            throw new ValidationException("Order has no items");
        
        await Task.CompletedTask;
    }
}

// Basic pipeline usage
public class OrderService
{
    private readonly IPipelineBuilder _pipelineBuilder;
    
    public OrderService(IPipelineBuilder pipelineBuilder)
    {
        _pipelineBuilder = pipelineBuilder;
    }
    
    public async Task ProcessOrderAsync(string orderId)
    {
        var pipeline = _pipelineBuilder
            .AddStage<ValidateOrderStage>()
            .AddStage<ProcessOrderStage>()
            .Build();
        
        var context = new OrderProcessingContext
        {
            OrderId = orderId,
            // Load order data
        };
        
        await pipeline.ExecuteAsync(context);
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

## Consumer Groups and Scaling

The .NET implementation provides full consumer group support for automatic scaling:

```csharp
// Configure consumer groups for automatic scaling
services.AddMmateMessaging()
    .WithConsumerGroups(groups =>
    {
        groups.AddGroup("order-processing", size: 5, maxConcurrency: 10)
              .WithQueue("order.commands")
              .WithPrefetchCount(20)
              .WithRetryPolicy(policy => policy
                  .MaxAttempts(3)
                  .WithExponentialBackoff(TimeSpan.FromSeconds(1)));
                  
        groups.AddGroup("payment-processing", size: 3, maxConcurrency: 5)
              .WithQueue("payment.commands")
              .WithPrefetchCount(10);
              
        groups.AddGroup("notification-sending", size: 10, maxConcurrency: 20)
              .WithQueue("notification.events")
              .WithPrefetchCount(50);
    });

// Handler with consumer group attributes
[ConsumerGroup("order-processing")]
[MessageHandler(Queue = "order.commands", PrefetchCount = 20)]
[RetryPolicy(MaxAttempts = 3, InitialDelayMs = 1000, Strategy = BackoffStrategy.Exponential)]
public class OrderHandler : IMessageHandler<ProcessOrderCommand>
{
    private readonly ILogger<OrderHandler> _logger;
    private readonly IOrderService _orderService;
    private readonly IMetricsCollector _metrics;
    
    public OrderHandler(
        ILogger<OrderHandler> logger,
        IOrderService orderService,
        IMetricsCollector metrics)
    {
        _logger = logger;
        _orderService = orderService;
        _metrics = metrics;
    }
    
    public async Task HandleAsync(ProcessOrderCommand command, MessageContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            _logger.LogInformation("Processing order {OrderId} in consumer group", command.OrderId);
            
            await _orderService.ProcessOrderAsync(command);
            
            _metrics.RecordMessageProcessed(typeof(ProcessOrderCommand).Name, 
                stopwatch.Elapsed, success: true);
                
            _logger.LogInformation("Successfully processed order {OrderId} in {Duration}ms", 
                command.OrderId, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _metrics.RecordMessageProcessed(typeof(ProcessOrderCommand).Name, 
                stopwatch.Elapsed, success: false);
                
            _logger.LogError(ex, "Failed to process order {OrderId}", command.OrderId);
            throw;
        }
    }
}

// Advanced consumer group with custom configuration
public class HighVolumeEventProcessor : BackgroundService
{
    private readonly IConsumerGroup _consumerGroup;
    private readonly ILogger<HighVolumeEventProcessor> _logger;

    public HighVolumeEventProcessor(
        IConsumerGroupFactory factory,
        ILogger<HighVolumeEventProcessor> logger)
    {
        _consumerGroup = factory.CreateGroup(new ConsumerGroupConfig
        {
            Name = "high-volume-events",
            Size = Environment.ProcessorCount * 2,
            MaxConcurrency = 50,
            QueueName = "events.high-volume",
            PrefetchCount = 100,
            AutoScaling = new AutoScalingConfig
            {
                Enabled = true,
                MinSize = 2,
                MaxSize = 20,
                ScaleUpThreshold = 0.8,   // Scale up at 80% utilization
                ScaleDownThreshold = 0.3, // Scale down at 30% utilization
                ScaleUpCooldown = TimeSpan.FromMinutes(2),
                ScaleDownCooldown = TimeSpan.FromMinutes(5)
            }
        });
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _consumerGroup.StartAsync(stoppingToken);
        
        // Monitor consumer group health
        while (!stoppingToken.IsCancellationRequested)
        {
            var metrics = await _consumerGroup.GetMetricsAsync();
            
            _logger.LogInformation(
                "Consumer group status: {ActiveConsumers}/{TotalConsumers} consumers, " +
                "Queue depth: {QueueDepth}, Utilization: {Utilization:P2}",
                metrics.ActiveConsumers, metrics.TotalConsumers,
                metrics.QueueDepth, metrics.Utilization);
                
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
    
    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        await _consumerGroup.StopAsync(cancellationToken);
        await base.StopAsync(cancellationToken);
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