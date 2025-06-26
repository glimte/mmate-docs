# Getting Started with Mmate

This guide will help you get up and running with Mmate quickly. We'll cover installation, basic configuration, and common usage patterns.

## Prerequisites

- .NET 9.0 SDK or later
- RabbitMQ server (for messaging transport)
- Visual Studio 2022, VS Code, or JetBrains Rider (optional)

## Installation

### Installing Mmate Packages

Mmate is modular - install only the components you need:

```bash
# Core messaging functionality
dotnet add package Mmate.Messaging

# Workflow engine (optional)
dotnet add package Mmate.StageFlow

# Message interceptors (optional)
dotnet add package Mmate.Interceptors

# Sync/Async bridge (optional)
dotnet add package Mmate.SyncAsyncBridge

# Schema validation (optional)
dotnet add package Mmate.Schema
```

### Installing Command Line Tools

```bash
# Monitoring tool
dotnet tool install --global Mmate.Monitor
```

## Quick Start Example

Here's a complete example to get you started:

### 1. Define Your Messages

```csharp
using Mmate.Contracts;

// Command - for actions that change state
public class CreateOrderCommand : BaseCommand
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
}

public class OrderItem
{
    public string ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal Price { get; set; }
}

// Event - for notifications about what happened
public class OrderCreatedEvent : BaseEvent
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
}

// Query and Response - for retrieving data
public class GetOrderQuery : BaseQuery<GetOrderResponse>
{
    public string OrderId { get; set; }
}

public class GetOrderResponse : BaseResponse
{
    public string OrderId { get; set; }
    public string Status { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### 2. Create Message Handlers

```csharp
using Mmate.Messaging;

public class CreateOrderHandler : IMessageHandler<CreateOrderCommand>
{
    private readonly ILogger<CreateOrderHandler> _logger;
    private readonly IMessagePublisher _publisher;
    private readonly IOrderRepository _repository;

    public CreateOrderHandler(
        ILogger<CreateOrderHandler> logger,
        IMessagePublisher publisher,
        IOrderRepository repository)
    {
        _logger = logger;
        _publisher = publisher;
        _repository = repository;
    }

    public async Task HandleAsync(CreateOrderCommand command, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", command.CustomerId);

        // Create the order
        var order = new Order
        {
            Id = Guid.NewGuid().ToString(),
            CustomerId = command.CustomerId,
            Items = command.Items,
            TotalAmount = command.TotalAmount,
            Status = "Created",
            CreatedAt = DateTime.UtcNow
        };

        // Save to database
        await _repository.SaveAsync(order, cancellationToken);

        // Publish event
        await _publisher.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.TotalAmount,
            CorrelationId = command.CorrelationId
        }, cancellationToken);

        _logger.LogInformation("Order {OrderId} created successfully", order.Id);
    }
}

public class GetOrderHandler : IMessageHandler<GetOrderQuery, GetOrderResponse>
{
    private readonly IOrderRepository _repository;

    public GetOrderHandler(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task<GetOrderResponse> HandleAsync(GetOrderQuery query, CancellationToken cancellationToken)
    {
        var order = await _repository.GetByIdAsync(query.OrderId, cancellationToken);
        
        if (order == null)
        {
            return new GetOrderResponse
            {
                Success = false,
                ErrorMessage = "Order not found"
            };
        }

        return new GetOrderResponse
        {
            Success = true,
            OrderId = order.Id,
            Status = order.Status,
            CreatedAt = order.CreatedAt
        };
    }
}
```

### 3. Configure Services

```csharp
// In Program.cs or Startup.cs

var builder = WebApplication.CreateBuilder(args);

// Add Mmate messaging
builder.Services.AddMmateMessaging(options =>
{
    options.ConnectionString = "amqp://guest:guest@localhost:5672/";
    options.ExchangeName = "my-app";
    options.QueuePrefix = "myapp.";
})
.WithRabbitMqTransport(options =>
{
    // Connection resilience
    options.EnableAutomaticRecovery = true;
    options.MaxRetryAttempts = 5;
    options.InitialRetryDelayMs = 1000;
    
    // Circuit breaker
    options.EnableCircuitBreaker = true;
    options.CircuitBreakerFailureThreshold = 5;
    options.CircuitBreakerOpenTimeoutSeconds = 30;
})
.AddMessageHandler<CreateOrderHandler, CreateOrderCommand>()
.AddMessageHandler<GetOrderHandler, GetOrderQuery>();

// Add your other services
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

var app = builder.Build();

// Start message processing
app.Run();
```

### 4. Publishing Messages

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMessageDispatcher _dispatcher;

    public OrdersController(IMessageDispatcher dispatcher)
    {
        _dispatcher = dispatcher;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
    {
        // Send command
        await _dispatcher.PublishAsync(new CreateOrderCommand
        {
            CustomerId = request.CustomerId,
            Items = request.Items,
            TotalAmount = request.TotalAmount,
            CorrelationId = HttpContext.TraceIdentifier
        });

        return Accepted();
    }

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrder(string orderId)
    {
        // Send query and wait for response
        var response = await _dispatcher.RequestAsync<GetOrderQuery, GetOrderResponse>(
            new GetOrderQuery { OrderId = orderId },
            timeout: TimeSpan.FromSeconds(30));

        if (!response.Success)
        {
            return NotFound(response.ErrorMessage);
        }

        return Ok(response);
    }
}
```

## Working with Interceptors

Add cross-cutting concerns without modifying your handlers:

```csharp
// 1. Create an interceptor
public class LoggingInterceptor : MessageInterceptorBase
{
    private readonly ILogger<LoggingInterceptor> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor> logger)
    {
        _logger = logger;
    }

    public override Task OnBeforeConsumeAsync(MessageContext context, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Processing {MessageType} with ID {MessageId}",
            context.Message.GetType().Name,
            context.GetHeader<string>("MessageId"));
        
        return Task.CompletedTask;
    }

    public override Task OnAfterConsumeAsync(MessageContext context, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Completed processing {MessageType}",
            context.Message.GetType().Name);
        
        return Task.CompletedTask;
    }
}

// 2. Register the interceptor
builder.Services.AddMmateInterceptors(interceptors =>
{
    interceptors.Add<LoggingInterceptor>();
});
```

## Using StageFlow for Workflows

Create multi-stage workflows for complex business processes:

```csharp
// 1. Define workflow state
public class OrderFulfillmentState
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public string PaymentId { get; set; }
    public string ShipmentId { get; set; }
    public string Status { get; set; }
}

// 2. Configure workflow
builder.Services.AddStageFlow(options =>
{
    options.StageQueuePrefix = "myapp.workflow.";
    options.MaxStageConcurrency = 5;
});

// 3. Define workflow stages
public class OrderFulfillmentWorkflow
{
    private readonly IFlowEndpointFactory _factory;

    public OrderFulfillmentWorkflow(IFlowEndpointFactory factory)
    {
        _factory = factory;
        ConfigureWorkflow();
    }

    private void ConfigureWorkflow()
    {
        var workflow = _factory.CreateFlow<CreateOrderCommand, OrderFulfillmentState>("OrderFulfillment");

        // Stage 1: Validate and initialize
        workflow.Stage<CreateOrderCommand>(async (context, state, command) =>
        {
            state.OrderId = Guid.NewGuid().ToString();
            state.CustomerId = command.CustomerId;
            state.Items = command.Items;
            state.Status = "Validating";

            // Request inventory check
            await context.RequestAsync("inventory.check", new CheckInventoryRequest
            {
                Items = command.Items
            });
        });

        // Stage 2: Process payment
        workflow.Stage<CheckInventoryResponse>(async (context, state, response) =>
        {
            if (!response.AllItemsAvailable)
            {
                state.Status = "Failed - Inventory Unavailable";
                return;
            }

            state.Status = "Processing Payment";

            // Request payment
            await context.RequestAsync("payment.process", new ProcessPaymentRequest
            {
                OrderId = state.OrderId,
                Amount = state.Items.Sum(i => i.Price * i.Quantity)
            });
        });

        // Stage 3: Create shipment
        workflow.Stage<ProcessPaymentResponse>(async (context, state, response) =>
        {
            if (!response.Success)
            {
                state.Status = "Failed - Payment Declined";
                return;
            }

            state.PaymentId = response.PaymentId;
            state.Status = "Creating Shipment";

            // Request shipment
            await context.RequestAsync("shipment.create", new CreateShipmentRequest
            {
                OrderId = state.OrderId,
                Items = state.Items,
                CustomerId = state.CustomerId
            });
        });

        // Final stage: Complete order
        workflow.LastStage<CreateShipmentResponse, OrderFulfillmentResult>(async (context, state, response) =>
        {
            state.ShipmentId = response.ShipmentId;
            state.Status = "Completed";

            // Publish completion event
            await context.PublishAsync(new OrderFulfilledEvent
            {
                OrderId = state.OrderId,
                CustomerId = state.CustomerId,
                ShipmentId = state.ShipmentId
            });

            return new OrderFulfillmentResult
            {
                Success = true,
                OrderId = state.OrderId,
                Status = state.Status,
                ShipmentId = state.ShipmentId
            };
        });
    }
}
```

## Error Handling Best Practices

### 1. Configure Retry Policies

```csharp
builder.Services.AddMmateMessaging()
    .ConfigureRetryPolicy(options =>
    {
        // Customize retry attempts by error type
        options.ConnectionFailureMaxRetries = 5;
        options.TimeoutMaxRetries = 3;
        options.ValidationFailureMaxRetries = 0; // Don't retry validation errors
        
        // Backoff settings
        options.InitialDelayMs = 1000;
        options.MaxDelayMs = 30000;
        options.BackoffMultiplier = 2.0;
        options.JitterEnabled = true;
    });
```

### 2. Handle Dead Letter Queue

```csharp
public class DeadLetterHandler : IMessageHandler<DeadLetterMessage>
{
    private readonly ILogger<DeadLetterHandler> _logger;
    private readonly INotificationService _notifications;

    public async Task HandleAsync(DeadLetterMessage message, CancellationToken cancellationToken)
    {
        _logger.LogError("Message {MessageId} moved to DLQ after {Attempts} attempts. Error: {Error}",
            message.OriginalMessageId,
            message.RetryCount,
            message.LastError);

        // Send alert
        await _notifications.SendAlertAsync(
            $"Message processing failed: {message.OriginalMessageType}",
            cancellationToken);
    }
}
```

### 3. Monitor Circuit Breaker State

```csharp
public class HealthCheckService : IHealthCheck
{
    private readonly RabbitMqConnectionManager _connectionManager;

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var state = _connectionManager.CircuitBreakerState;
        
        return state switch
        {
            CircuitBreakerState.Open => Task.FromResult(
                HealthCheckResult.Unhealthy("Circuit breaker is open")),
            CircuitBreakerState.HalfOpen => Task.FromResult(
                HealthCheckResult.Degraded("Circuit breaker is half-open")),
            _ => Task.FromResult(HealthCheckResult.Healthy())
        };
    }
}
```

## Monitoring Your Application

Use the Mmate.Monitor CLI tool to monitor your queues:

```bash
# List all queues
mmate-monitor queue list -c "amqp://guest:guest@localhost:5672/"

# Watch a specific queue
mmate-monitor watch myapp.orders -c "amqp://guest:guest@localhost:5672/"

# Check dead letter queues
mmate-monitor dlq list -c "amqp://guest:guest@localhost:5672/"

# Requeue messages from DLQ
mmate-monitor dlq requeue myapp.orders.dlq -c "amqp://guest:guest@localhost:5672/"
```

## Testing Your Handlers

```csharp
[Fact]
public async Task CreateOrderHandler_Should_PublishEvent_When_OrderCreated()
{
    // Arrange
    var logger = new Mock<ILogger<CreateOrderHandler>>();
    var publisher = new Mock<IMessagePublisher>();
    var repository = new Mock<IOrderRepository>();
    
    var handler = new CreateOrderHandler(logger.Object, publisher.Object, repository.Object);
    var command = new CreateOrderCommand
    {
        CustomerId = "customer-123",
        Items = new List<OrderItem> { /* ... */ },
        TotalAmount = 100.00m
    };

    // Act
    await handler.HandleAsync(command, CancellationToken.None);

    // Assert
    repository.Verify(r => r.SaveAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Once);
    publisher.Verify(p => p.PublishAsync(
        It.Is<OrderCreatedEvent>(e => e.CustomerId == command.CustomerId),
        It.IsAny<CancellationToken>()), Times.Once);
}
```

## Next Steps

1. Explore the [API Reference](./API-REFERENCE.md) for detailed documentation
2. Check out [Examples](./examples/) for more complex scenarios
3. Learn about [StageFlow Workflows](./stageflow/README.md)
4. Understand [Message Interceptors](./interceptors/README.md)
5. Configure [Monitoring and Health Checks](./monitoring/README.md)

## Common Issues and Solutions

### Connection Refused
- Ensure RabbitMQ is running: `docker run -d -p 5672:5672 -p 15672:15672 rabbitmq:3-management`
- Check connection string format: `amqp://user:pass@host:port/vhost`

### Messages Not Being Processed
- Verify handlers are registered: `.AddMessageHandler<THandler, TMessage>()`
- Check queue names match your configuration
- Ensure the application is running (not just built)

### Circuit Breaker Opens Frequently
- Increase failure threshold in configuration
- Check network stability
- Verify RabbitMQ server resources

### Performance Issues
- Adjust `MaxConcurrentMessages` for parallel processing
- Use batching for high-volume scenarios
- Monitor queue depths and consumer counts