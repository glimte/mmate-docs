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
// Create mmate client with auto-queue creation
client, err := mmate.NewClientWithOptions("amqp://admin:admin@localhost:5672/",
    mmate.WithServiceName("order-service"),
    mmate.WithQueueBindings(
        messaging.QueueBinding{
            Exchange:   "mmate.events",
            RoutingKey: "order.service.*",
        },
    ),
)
if err != nil {
    log.Fatalf("Failed to create client: %v", err)
}
defer client.Close()

// Get publisher from client
publisher := client.Publisher()

// Define your message types
type OrderCreatedEvent struct {
    contracts.BaseMessage
    OrderID    string  `json:"orderId"`
    CustomerID string  `json:"customerId"`
    Amount     float64 `json:"amount"`
}

// Register message types for deserialization
messaging.Register("OrderCreatedEvent", func() contracts.Message { 
    return &OrderCreatedEvent{} 
})

// Publish event
orderEvent := &OrderCreatedEvent{}
orderEvent.Type = "OrderCreatedEvent"
orderEvent.ID = "ORDER-123"
orderEvent.CorrelationID = "corr-" + time.Now().Format("20060102150405")
orderEvent.Timestamp = time.Now()
orderEvent.OrderID = "ORDER-123"
orderEvent.CustomerID = "CUST-456"
orderEvent.Amount = 99.99

err = publisher.Publish(ctx, orderEvent,
    messaging.WithExchange("mmate.events"),
    messaging.WithRoutingKey("order.service.created"),
    messaging.WithPersistent(true),
)

// Publish with options
err = publisher.Publish(ctx, orderEvent,
    messaging.WithExchange("mmate.events"),
    messaging.WithRoutingKey("orders.created.priority"),
    messaging.WithReplyTo("order.service.reply"),
    messaging.WithPersistent(true),
)
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
// Get subscriber and dispatcher from client
subscriber := client.Subscriber()
dispatcher := client.Dispatcher()

// Define message handler
orderHandler := messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
    event, ok := msg.(*OrderCreatedEvent)
    if !ok {
        log.Printf("❌ Unexpected message type: %T", msg)
        return nil
    }
    
    log.Printf("📦 Processing Order: %s for Customer: %s", 
        event.OrderID, event.CustomerID)
    
    // Process the order
    return orderService.ProcessOrder(ctx, event.OrderID)
})

// Register handler with dispatcher
err = dispatcher.RegisterHandler(&OrderCreatedEvent{}, orderHandler)
if err != nil {
    log.Fatalf("Failed to register handler: %v", err)
}

// Subscribe to the auto-created service queue
err = subscriber.Subscribe(ctx, client.ServiceQueue(), "OrderCreatedEvent", dispatcher,
    messaging.WithAutoAck(true), // Enable auto-acknowledgment
)
if err != nil {
    log.Fatalf("Failed to subscribe: %v", err)
}

log.Printf("✅ Subscribed to queue: %s", client.ServiceQueue())

// Subscribe with typed handler struct
type OrderHandler struct {
    orderService OrderService
    logger       *log.Logger
}

func (h *OrderHandler) Handle(ctx context.Context, msg contracts.Message) error {
    event := msg.(*OrderCreatedEvent)
    
    h.logger.Printf("Processing order %s", event.OrderID)
    return h.orderService.ProcessOrder(ctx, event.OrderID)
}

// Register struct handler
orderHandler := &OrderHandler{
    orderService: orderService,
    logger:       log.Default(),
}
err = dispatcher.RegisterHandler(&OrderCreatedEvent{}, orderHandler)
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
    client.ServiceQueue(),
    "OrderEvent",
    dispatcher,
    messaging.WithAutoAck(false),      // Manual acknowledgment
    messaging.WithPrefetchCount(10),   // Process 10 messages at once
)

// Subscribe to specific routing patterns
err = subscriber.Subscribe(ctx,
    client.ServiceQueue(),
    "OrderEvent", 
    dispatcher,
    messaging.WithAutoAck(true),
)
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

> **Go Implementation Note**: All advanced features are available through both the `Client` convenience methods and the `Publisher` directly. Using the Client methods is recommended for better discoverability.

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
// Using Client convenience methods (recommended)
client, _ := mmate.NewClient(connectionString)

// Batch publish
batch := client.NewBatch()

for _, order := range orders {
    batch.Add(&OrderCreatedEvent{
        BaseEvent: contracts.NewBaseEvent(
            "OrderCreatedEvent", order.ID),
        OrderID: order.ID,
    })
}

err := batch.Publish(ctx)

// Or using publisher directly
batch := client.Publisher().NewBatch()
// ... same as above

// Transactional publish
tx, err := client.BeginTx()
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

> **Plugin Requirement**: Message scheduling requires the [RabbitMQ Delayed Message Exchange plugin](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange) to be installed and enabled.

#### What is the Delayed Message Exchange Plugin?

The plugin adds the ability to delay message delivery in RabbitMQ:
- **Without plugin**: Messages are delivered immediately to consumers
- **With plugin**: Messages can be scheduled for future delivery (e.g., "deliver in 5 minutes" or "deliver at 3:00 PM")

#### Installation

The plugin must be installed **on the RabbitMQ server**, not in your application:

**Option 1: Using Docker Compose**
```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    command: >
      bash -c "
        rabbitmq-plugins enable rabbitmq_delayed_message_exchange &&
        rabbitmq-server
      "
```

**Option 2: Using Dockerfile**
```dockerfile
FROM rabbitmq:3-management
RUN rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

**Option 3: Local RabbitMQ Installation**
```bash
# On the server where RabbitMQ is running
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
# Then restart RabbitMQ
systemctl restart rabbitmq-server  # or your system's restart command
```

> **Important**: If the plugin isn't installed, scheduled messages will be delivered immediately (the delay is ignored)

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
// Using Client convenience methods (recommended)
client, _ := mmate.NewClient(connectionString)

// Delayed message
err = client.PublishWithDelay(ctx,
    command,
    5*time.Minute)

// Scheduled message
err = client.ScheduleCommand(ctx,
    command,
    time.Now().Add(time.Hour))

// Or using publisher directly
err = client.Publisher().ScheduleCommand(ctx,
    command,
    time.Now().Add(time.Hour))

// Recurring messages (not yet implemented)
// Requires additional scheduler service
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

## FIFO Queues

FIFO (First-In-First-Out) queues guarantee message ordering and prevent duplicate processing using RabbitMQ's single-active-consumer feature.

### When to Use FIFO Queues

- **Financial transactions**: Ensure deposits/withdrawals are processed in order
- **State machines**: Sequential state transitions must maintain order
- **Audit trails**: Events must be recorded in the correct sequence
- **Critical business processes**: When message ordering is essential

### Go Implementation

```go
// Enable FIFO mode when creating client
client, err := mmate.NewClientWithOptions("amqp://admin:admin@localhost:5672/",
    mmate.WithServiceName("banking-service"),
    mmate.WithFIFOMode(true), // Enable FIFO queue with single-active-consumer
    mmate.WithQueueBindings(
        messaging.QueueBinding{
            Exchange:   "mmate.events",
            RoutingKey: "banking.service.*",
        },
    ),
)

// Banking request message with correlation tracking
type BankingRequest struct {
    contracts.BaseMessage
    Operation   string  `json:"operation"`
    FromAccount string  `json:"fromAccount"`
    ToAccount   string  `json:"toAccount"`
    Amount      float64 `json:"amount"`
    RequestID   string  `json:"requestId"`
}

// Handler processes messages in FIFO order
bankingHandler := messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
    request := msg.(*BankingRequest)
    
    log.Printf("🏦 Processing %s: $%.2f (Correlation: %s)", 
        request.Operation, request.Amount, request.CorrelationID)
    
    // Process banking operation
    return processBankingOperation(request)
})

// Subscribe with auto-ack for FIFO processing
err = subscriber.Subscribe(ctx, client.ServiceQueue(), "BankingRequest", dispatcher,
    messaging.WithAutoAck(true), // Safe with idempotent operations
)
```

### FIFO vs Multiple Consumers

**FIFO Queues:**
- ✅ Guaranteed message ordering
- ✅ No duplicate processing (single-active-consumer)
- ⚠️ Limited horizontal scaling per queue

**Multiple Consumer Strategy:**
```go
// For K8s deployments - multiple services can consume from FIFO queues
// Each service gets its own FIFO queue, messages distributed via routing

// Container B: banking-service-1
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("banking-service-1"),
    mmate.WithFIFOMode(true),
    mmate.WithQueueBindings(
        messaging.QueueBinding{
            Exchange:   "mmate.events", 
            RoutingKey: "banking.service.*", // Receives all banking.service.* messages
        },
    ),
)

// Container C: banking-service-2 
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("banking-service-2"), 
    mmate.WithFIFOMode(true),
    mmate.WithQueueBindings(
        messaging.QueueBinding{
            Exchange:   "mmate.events",
            RoutingKey: "banking.service.*", // Also receives all banking.service.* messages
        },
    ),
)

// Publisher sends to both using wildcard routing
err = publisher.Publish(ctx, bankingRequest,
    messaging.WithExchange("mmate.events"),
    messaging.WithRoutingKey("banking.service.transfer"), // Reaches both services
    messaging.WithReplyTo("client.reply"),
)
```

### Benefits with Idempotent Operations

When operations are idempotent, multiple FIFO consumers provide:
- **High Availability**: Service continues if one instance fails
- **Load Distribution**: Different services can process same requests safely
- **Fault Tolerance**: System resilience with redundancy
- **Order Guarantee**: Each service processes messages in correct order

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