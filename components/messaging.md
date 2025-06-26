# Messaging Component

The messaging component is the core of Mmate, providing publishers, subscribers, dispatchers, and handlers for all message patterns.

## Overview

The messaging component handles:
- Message publishing to RabbitMQ exchanges
- Message consumption from queues
- Message routing to handlers
- Connection and channel management
- Error handling and acknowledgments

## Architecture

```
┌─────────────────┐     ┌─────────────────┐
│   Publisher     │     │   Subscriber    │
├─────────────────┤     ├─────────────────┤
│ MessagePublisher│     │MessageSubscriber│
│ EventPublisher  │     │ EventSubscriber │
│ CommandPublisher│     │CommandSubscriber│
└────────┬────────┘     └────────┬────────┘
         │                       │
    ┌────┴───────────────────────┴────┐
    │        Transport Layer          │
    │    (RabbitMQ Connection)        │
    └─────────────────────────────────┘
```

## Publishing Messages

### Basic Publishing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Create publisher
var publisher = new MessagePublisher(
    connectionString: "amqp://localhost",
    logger: logger);

// Publish event
await publisher.PublishEventAsync(
    new OrderCreatedEvent 
    { 
        OrderId = "ORDER-123",
        CustomerId = "CUST-456",
        Amount = 99.99m
    });

// Publish command
await publisher.PublishCommandAsync(
    new ProcessOrderCommand 
    { 
        OrderId = "ORDER-123" 
    });

// Publish with options
await publisher.PublishEventAsync(
    orderEvent,
    options => options
        .WithRoutingKey("orders.created.priority")
        .WithHeader("priority", "high")
        .WithExpiration(TimeSpan.FromHours(1)));
```

</td>
<td>

```go
// Create publisher
publisher, err := messaging.NewMessagePublisher(
    rabbitmq.NewPublisher(channelPool),
    messaging.WithLogger(logger))

// Publish event
err = publisher.PublishEvent(ctx,
    &OrderCreatedEvent{
        BaseEvent:  contracts.NewBaseEvent(
            "OrderCreatedEvent", "ORDER-123"),
        OrderID:    "ORDER-123",
        CustomerID: "CUST-456", 
        Amount:     99.99,
    })

// Publish command
err = publisher.PublishCommand(ctx,
    &ProcessOrderCommand{
        BaseCommand: contracts.NewBaseCommand(
            "ProcessOrderCommand"),
        OrderID:     "ORDER-123",
    })

// Publish with options
err = publisher.PublishEvent(ctx,
    orderEvent,
    messaging.WithRoutingKey("orders.created.priority"),
    messaging.WithHeader("priority", "high"),
    messaging.WithExpiration(time.Hour))
```

</td>
</tr>
</table>

### Publishing Options

| Option | Description | Example |
|--------|-------------|---------|
| `WithRoutingKey` | Set custom routing key | `"orders.created.us-west"` |
| `WithHeader` | Add custom header | `"userId", "12345"` |
| `WithExpiration` | Message TTL | `1 hour` |
| `WithPriority` | Message priority (0-9) | `5` |
| `WithPersistent` | Persist to disk | `true` |

## Consuming Messages

### Basic Subscription

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Create subscriber
var subscriber = new MessageSubscriber(
    connectionString: "amqp://localhost",
    logger: logger);

// Subscribe to events
await subscriber.SubscribeAsync<OrderCreatedEvent>(
    queueName: "order-processor",
    handler: async (message, context) =>
    {
        await orderService.ProcessOrder(
            message.OrderId);
    });

// Subscribe with typed handler
public class OrderHandler : 
    IMessageHandler<OrderCreatedEvent>
{
    public async Task HandleAsync(
        OrderCreatedEvent message,
        MessageContext context)
    {
        await ProcessOrder(message);
    }
}

await subscriber.SubscribeAsync<OrderCreatedEvent>(
    "order-processor",
    new OrderHandler());
```

</td>
<td>

```go
// Create subscriber
subscriber, err := messaging.NewMessageSubscriber(
    rabbitmq.NewSubscriber(channelPool),
    messaging.WithLogger(logger))

// Subscribe to events
err = subscriber.Subscribe(ctx,
    "order-processor",
    "OrderCreatedEvent",
    messaging.HandlerFunc(func(
        ctx context.Context,
        msg contracts.Message) error {
        
        event := msg.(*OrderCreatedEvent)
        return orderService.ProcessOrder(
            ctx, event.OrderID)
    }))

// Subscribe with typed handler
type OrderHandler struct {
    orderService OrderService
}

func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    event := msg.(*OrderCreatedEvent)
    return h.orderService.ProcessOrder(
        ctx, event.OrderID)
}

err = subscriber.Subscribe(ctx,
    "order-processor",
    "OrderCreatedEvent", 
    &OrderHandler{orderService: svc})
```

</td>
</tr>
</table>

### Subscription Options

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Subscribe with options
await subscriber.SubscribeAsync<OrderEvent>(
    "order-processor",
    handler,
    options => options
        .WithPrefetchCount(10)
        .WithBindingKey("orders.*.created")
        .WithAutoAck(false)
        .WithDurable(true)
        .WithExclusive(false));

// Consumer group
await subscriber.SubscribeAsync<OrderEvent>(
    "order-processor",
    handler,
    options => options
        .WithConsumerGroup("order-workers", 5)
        .WithPrefetchCount(2));
```

</td>
<td>

```go
// Subscribe with options
err = subscriber.Subscribe(ctx,
    "order-processor",
    "OrderEvent",
    handler,
    messaging.WithPrefetchCount(10),
    messaging.WithBindingKey("orders.*.created"),
    messaging.WithAutoAck(false),
    messaging.WithDurable(true),
    messaging.WithExclusive(false))

// Consumer group
group := messaging.NewConsumerGroup(
    "order-workers",
    messaging.WithGroupSize(5),
    messaging.WithPrefetchCount(2))

err = group.Subscribe(ctx,
    "order-processor",
    handler)
```

</td>
</tr>
</table>

## Message Handlers

### Handler Interface

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Basic handler interface
public interface IMessageHandler<TMessage>
{
    Task HandleAsync(
        TMessage message,
        MessageContext context);
}

// Request handler (with reply)
public interface IRequestHandler<TRequest, TReply>
{
    Task<TReply> HandleAsync(
        TRequest request,
        RequestContext context);
}

// Handler with dependencies
public class OrderHandler : 
    IMessageHandler<CreateOrderCommand>
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrderHandler> _logger;
    
    public OrderHandler(
        IOrderService orderService,
        ILogger<OrderHandler> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }
    
    public async Task HandleAsync(
        CreateOrderCommand command,
        MessageContext context)
    {
        _logger.LogInformation(
            "Processing order {OrderId}",
            command.OrderId);
            
        await _orderService.CreateOrder(
            command);
    }
}
```

</td>
<td>

```go
// Basic handler interface
type Handler interface {
    Handle(ctx context.Context, 
        msg contracts.Message) error
}

// Handler function adapter
type HandlerFunc func(
    ctx context.Context,
    msg contracts.Message) error

func (f HandlerFunc) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    return f(ctx, msg)
}

// Handler with dependencies
type OrderHandler struct {
    orderService OrderService
    logger       *slog.Logger
}

func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CreateOrderCommand)
    
    h.logger.Info("Processing order",
        "orderId", cmd.OrderID)
    
    return h.orderService.CreateOrder(
        ctx, cmd)
}

// Request handler (with reply)
type RequestHandler struct {
    publisher messaging.Publisher
}

func (h *RequestHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    query := msg.(*GetOrderQuery)
    order, err := h.getOrder(query.OrderID)
    if err != nil {
        return err
    }
    
    reply := &OrderReply{
        BaseReply: contracts.NewBaseReply(
            query.GetID(), 
            query.GetCorrelationID()),
        Order: order,
    }
    
    return h.publisher.PublishReply(
        ctx, reply, query.ReplyTo)
}
```

</td>
</tr>
</table>

### Error Handling

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderHandler : 
    IMessageHandler<CreateOrderCommand>
{
    public async Task HandleAsync(
        CreateOrderCommand command,
        MessageContext context)
    {
        try
        {
            await ProcessOrder(command);
        }
        catch (ValidationException ex)
        {
            // Permanent failure - don't retry
            throw new PermanentFailureException(
                "Invalid order", ex);
        }
        catch (DatabaseException ex)
        {
            // Transient failure - retry
            throw new TransientFailureException(
                "Database error", ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Unexpected error");
            throw;
        }
    }
}
```

</td>
<td>

```go
type OrderHandler struct{}

func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CreateOrderCommand)
    
    err := h.processOrder(ctx, cmd)
    if err != nil {
        // Check error type
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            // Permanent failure - don't retry
            return messaging.NewPermanentError(
                "invalid order", err)
        }
        
        var dbErr *DatabaseError
        if errors.As(err, &dbErr) {
            // Transient failure - retry
            return messaging.NewTransientError(
                "database error", err)
        }
        
        // Unknown error
        return fmt.Errorf(
            "processing failed: %w", err)
    }
    
    return nil
}
```

</td>
</tr>
</table>

## Message Dispatcher

The dispatcher routes messages to appropriate handlers based on message type.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure dispatcher
services.AddMmateDispatcher(dispatcher =>
{
    // Register handlers
    dispatcher.RegisterHandler<CreateOrderCommand>(
        new CreateOrderHandler());
        
    dispatcher.RegisterHandler<OrderCreatedEvent>(
        new OrderEventHandler());
        
    // Register by assembly
    dispatcher.RegisterHandlersFromAssembly(
        typeof(OrderHandler).Assembly);
});

// Manual registration
var dispatcher = new MessageDispatcher();
dispatcher.Register<CreateOrderCommand>(
    async (cmd, ctx) => 
    {
        await orderService.CreateOrder(cmd);
    });
```

</td>
<td>

```go
// Create dispatcher
dispatcher := messaging.NewDispatcher()

// Register handlers
dispatcher.Register("CreateOrderCommand",
    &CreateOrderHandler{})
    
dispatcher.Register("OrderCreatedEvent",
    &OrderEventHandler{})

// Register with factory
dispatcher.RegisterFactory(
    "ProcessPaymentCommand",
    func() messaging.Handler {
        return &PaymentHandler{
            service: paymentService,
        }
    })

// Function handler
dispatcher.RegisterFunc(
    "UpdateInventoryCommand",
    func(ctx context.Context, 
        msg contracts.Message) error {
        
        cmd := msg.(*UpdateInventoryCommand)
        return inventory.Update(
            ctx, cmd.ProductID, cmd.Quantity)
    })
```

</td>
</tr>
</table>

## Connection Management

### Connection Options

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure connection
services.AddMmate(options =>
{
    options.ConnectionString = 
        "amqp://user:pass@localhost:5672/vhost";
    
    // Connection pool
    options.MaxConnections = 5;
    options.MaxChannelsPerConnection = 10;
    
    // Timeouts
    options.ConnectionTimeout = 
        TimeSpan.FromSeconds(30);
    options.RequestTimeout = 
        TimeSpan.FromSeconds(60);
    
    // Retry
    options.EnableAutoRecovery = true;
    options.NetworkRecoveryInterval = 
        TimeSpan.FromSeconds(10);
});
```

</td>
<td>

```go
// Create connection manager
connManager := rabbitmq.NewConnectionManager(
    "amqp://user:pass@localhost:5672/vhost",
    rabbitmq.WithMaxConnections(5),
    rabbitmq.WithConnectionTimeout(30*time.Second),
    rabbitmq.WithAutoRecovery(true),
    rabbitmq.WithRecoveryInterval(10*time.Second))

// Create channel pool
channelPool, err := rabbitmq.NewChannelPool(
    connManager,
    rabbitmq.WithMaxChannels(10),
    rabbitmq.WithChannelTimeout(60*time.Second))
```

</td>
</tr>
</table>

### Health Monitoring

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Add health checks
services.AddHealthChecks()
    .AddRabbitMQ(
        rabbitConnectionString: connectionString,
        name: "rabbitmq",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "messaging", "rabbitmq" });

// Custom health check
public class MessagingHealthCheck : IHealthCheck
{
    public async Task<HealthCheckResult> 
        CheckHealthAsync(
            HealthCheckContext context,
            CancellationToken ct)
    {
        try
        {
            var isHealthy = await 
                _messaging.CheckHealthAsync(ct);
                
            return isHealthy
                ? HealthCheckResult.Healthy()
                : HealthCheckResult.Unhealthy(
                    "Connection failed");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Check failed", ex);
        }
    }
}
```

</td>
<td>

```go
// Health check interface
type HealthChecker struct {
    connManager *rabbitmq.ConnectionManager
}

func (h *HealthChecker) CheckHealth(
    ctx context.Context) error {
    
    // Check connection
    conn, err := h.connManager.GetConnection()
    if err != nil {
        return fmt.Errorf(
            "connection unavailable: %w", err)
    }
    
    // Check channel
    ch, err := conn.Channel()
    if err != nil {
        return fmt.Errorf(
            "channel creation failed: %w", err)
    }
    defer ch.Close()
    
    // Verify exchange exists
    err = ch.ExchangeDeclarePassive(
        "health.check", "topic",
        false, false, false, false, nil)
    if err != nil {
        return fmt.Errorf(
            "exchange check failed: %w", err)
    }
    
    return nil
}

// HTTP endpoint
func (h *HealthChecker) ServeHTTP(
    w http.ResponseWriter, 
    r *http.Request) {
    
    ctx := r.Context()
    if err := h.CheckHealth(ctx); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]any{
            "status": "unhealthy",
            "error":  err.Error(),
        })
        return
    }
    
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]any{
        "status": "healthy",
    })
}
```

</td>
</tr>
</table>

## Advanced Features

### Batch Publishing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Batch publish
var batch = publisher.CreateBatch();

foreach (var order in orders)
{
    batch.Add(new OrderCreatedEvent 
    { 
        OrderId = order.Id 
    });
}

await batch.PublishAsync();

// Transactional publish
using var tx = await publisher
    .BeginTransactionAsync();
    
try
{
    await tx.PublishEventAsync(orderCreated);
    await tx.PublishCommandAsync(processPayment);
    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

</td>
<td>

```go
// Batch publish
batch := publisher.NewBatch()

for _, order := range orders {
    batch.Add(&OrderCreatedEvent{
        BaseEvent: contracts.NewBaseEvent(
            "OrderCreatedEvent", order.ID),
        OrderID: order.ID,
    })
}

err := batch.Publish(ctx)

// Transactional publish
tx, err := publisher.BeginTx()
if err != nil {
    return err
}
defer tx.Rollback()

err = tx.PublishEvent(ctx, orderCreated)
if err != nil {
    return err
}

err = tx.PublishCommand(ctx, processPayment)
if err != nil {
    return err
}

return tx.Commit()
```

</td>
</tr>
</table>

### Message Scheduling

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Delayed message
await publisher.PublishCommandAsync(
    command,
    options => options
        .WithDelay(TimeSpan.FromMinutes(5)));

// Scheduled message
await publisher.ScheduleCommandAsync(
    command,
    scheduledFor: DateTime.UtcNow.AddHours(1));

// Recurring messages
await publisher.ScheduleRecurringAsync(
    new HealthCheckCommand(),
    cronExpression: "0 */5 * * * *"); // Every 5 min
```

</td>
<td>

```go
// Delayed message
err = publisher.PublishCommand(ctx,
    command,
    messaging.WithDelay(5*time.Minute))

// Scheduled message
err = publisher.ScheduleCommand(ctx,
    command,
    time.Now().Add(time.Hour))

// Recurring messages
err = publisher.ScheduleRecurring(ctx,
    &HealthCheckCommand{},
    "0 */5 * * * *") // Every 5 min
```

</td>
</tr>
</table>

## Performance Tuning

### Publisher Performance

1. **Connection Pooling**
   - Use multiple connections for high throughput
   - Balance channels across connections
   - Monitor channel utilization

2. **Batching**
   - Batch small messages together
   - Use publisher confirms for reliability
   - Consider transaction overhead

3. **Message Size**
   - Keep messages small (<64KB ideal)
   - Use references for large data
   - Compress if necessary

### Consumer Performance

1. **Prefetch Count**
   - Set based on processing time
   - Higher for fast processing
   - Lower for slow/CPU intensive

2. **Concurrency**
   - Use consumer groups for scaling
   - Balance workers with queue depth
   - Monitor consumer utilization

3. **Acknowledgment Strategy**
   - Auto-ack for non-critical messages
   - Manual ack for important data
   - Batch acknowledgments when possible

## Error Handling Strategies

### Retry Patterns

1. **Immediate Retry**
   - For transient network errors
   - Limited attempts (3-5)
   - No delay

2. **Exponential Backoff**
   - For rate limiting
   - Increasing delays
   - Jitter to prevent thundering herd

3. **Dead Letter Queue**
   - For poison messages
   - Manual intervention required
   - Preserve message for analysis

### Circuit Breaker

Prevent cascading failures:
- Open: Fail fast
- Half-open: Test recovery
- Closed: Normal operation

## Monitoring and Metrics

### Key Metrics

1. **Publisher Metrics**
   - Messages published/sec
   - Publish latency
   - Failed publishes
   - Connection status

2. **Consumer Metrics**
   - Messages consumed/sec
   - Processing time
   - Queue depth
   - Consumer lag

3. **System Metrics**
   - Active connections
   - Channel count
   - Memory usage
   - Error rates

## Best Practices

1. **Message Design**
   - Include all necessary data
   - Use correlation IDs
   - Version your messages
   - Keep payloads small

2. **Error Handling**
   - Distinguish permanent vs transient
   - Use appropriate retry strategies
   - Monitor dead letter queues
   - Log with context

3. **Performance**
   - Tune prefetch counts
   - Use connection pooling
   - Monitor queue depths
   - Scale based on metrics

4. **Reliability**
   - Use durable queues for critical data
   - Implement health checks
   - Plan for connection failures
   - Test failure scenarios

## Next Steps

- Learn about [Message Patterns](../../patterns/README.md)
- Explore [StageFlow](../stageflow/README.md) for workflows
- Implement [Interceptors](../interceptors/README.md)
- Add [Monitoring](../monitoring/README.md)