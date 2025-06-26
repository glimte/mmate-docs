# Messaging Patterns in Mmate

Mmate supports several messaging patterns that solve common distributed system challenges. This guide covers each pattern with examples in both .NET and Go.

## Pattern Overview

| Pattern | Description | Use Case | Delivery |
|---------|-------------|----------|----------|
| [Publish/Subscribe](#publishsubscribe) | Broadcast events to multiple consumers | Event notifications | One-to-many |
| [Request/Reply](#requestreply) | Get responses to requests | Service calls | One-to-one |
| [Work Queue](#work-queue) | Distribute tasks among workers | Load balancing | Competing consumers |
| [Topic Routing](#topic-routing) | Route by message properties | Filtered subscriptions | Selective |
| [Saga/Workflow](#sagaworkflow) | Multi-step processes | Order processing | Orchestrated |

## Publish/Subscribe

### Overview

Publishers emit events without knowing who will consume them. Multiple subscribers can receive the same event.

```
Publisher → Exchange → Queue 1 → Subscriber 1
                    ↘ Queue 2 → Subscriber 2
                    ↘ Queue 3 → Subscriber 3
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Define event
public class OrderCreatedEvent : Event
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public decimal Amount { get; set; }
}

// Publish
await publisher.PublishEventAsync(
    new OrderCreatedEvent
    {
        OrderId = "ORDER-123",
        CustomerId = "CUST-456",
        Amount = 99.99m
    });

// Subscribe
await subscriber.SubscribeAsync<OrderCreatedEvent>(
    "inventory.orders",
    async (evt, ctx) =>
    {
        await inventoryService
            .ReserveForOrder(evt.OrderId);
    });
```

</td>
<td>

```go
// Define event
type OrderCreatedEvent struct {
    contracts.BaseEvent
    OrderID    string  `json:"orderId"`
    CustomerID string  `json:"customerId"`
    Amount     float64 `json:"amount"`
}

// Publish
err := publisher.PublishEvent(ctx,
    &OrderCreatedEvent{
        BaseEvent:  contracts.NewBaseEvent(
            "OrderCreatedEvent", orderID),
        OrderID:    "ORDER-123",
        CustomerID: "CUST-456",
        Amount:     99.99,
    })

// Subscribe
err := subscriber.Subscribe(ctx,
    "inventory.orders",
    "OrderCreatedEvent",
    messaging.HandlerFunc(func(
        ctx context.Context,
        msg contracts.Message) error {
        
        evt := msg.(*OrderCreatedEvent)
        return inventoryService.
            ReserveForOrder(evt.OrderID)
    }))
```

</td>
</tr>
</table>

### When to Use

✅ **Good for:**
- Event notifications
- Decoupling services
- Activity feeds
- Audit logging
- Cache invalidation

❌ **Not for:**
- Expecting responses
- Guaranteed single processing
- Ordered processing (without additional work)

## Request/Reply

### Overview

Client sends a request and waits for a response, providing RPC-style communication over messaging.

```
Client → Request Queue → Server
Client ← Reply Queue   ← Server
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Define command and reply
public class CalculatePriceCommand : Command
{
    public string ProductId { get; set; }
    public int Quantity { get; set; }
}

public class PriceReply : Reply
{
    public decimal TotalPrice { get; set; }
    public decimal Discount { get; set; }
}

// Client - using bridge
var reply = await bridge
    .SendAndWaitAsync<PriceReply>(
        new CalculatePriceCommand
        {
            ProductId = "PROD-123",
            Quantity = 5
        },
        TimeSpan.FromSeconds(10));

Console.WriteLine($"Price: {reply.TotalPrice}");

// Server - handler
public class PriceHandler : 
    IRequestHandler<CalculatePriceCommand, PriceReply>
{
    public async Task<PriceReply> HandleAsync(
        CalculatePriceCommand cmd)
    {
        var price = await catalog
            .GetPrice(cmd.ProductId);
        
        return new PriceReply
        {
            TotalPrice = price * cmd.Quantity,
            Discount = CalculateDiscount(cmd)
        };
    }
}
```

</td>
<td>

```go
// Define command and reply
type CalculatePriceCommand struct {
    contracts.BaseCommand
    ProductID string `json:"productId"`
    Quantity  int    `json:"quantity"`
}

type PriceReply struct {
    contracts.BaseReply
    TotalPrice float64 `json:"totalPrice"`
    Discount   float64 `json:"discount"`
}

// Client - using bridge
reply, err := bridge.SendAndWait(ctx,
    &CalculatePriceCommand{
        BaseCommand: contracts.NewBaseCommand(
            "CalculatePriceCommand"),
        ProductID: "PROD-123",
        Quantity:  5,
    },
    "cmd.pricing.calculate",
    10*time.Second)

priceReply := reply.(*PriceReply)
fmt.Printf("Price: %.2f\n", priceReply.TotalPrice)

// Server - handler
type PriceHandler struct {
    catalog PriceCatalog
}

func (h *PriceHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CalculatePriceCommand)
    price := h.catalog.GetPrice(cmd.ProductID)
    
    reply := &PriceReply{
        BaseReply:  contracts.NewBaseReply(
            cmd.GetID(), cmd.GetCorrelationID()),
        TotalPrice: price * float64(cmd.Quantity),
        Discount:   h.calculateDiscount(cmd),
    }
    
    return h.publisher.PublishReply(ctx,
        reply, cmd.ReplyTo)
}
```

</td>
</tr>
</table>

### When to Use

✅ **Good for:**
- Service-to-service calls
- Query operations
- Commands needing confirmation
- Synchronous workflows

❌ **Not for:**
- Fire-and-forget operations
- Broadcasting to multiple services
- Very long-running operations

## Work Queue

### Overview

Distribute tasks among multiple workers for parallel processing and load balancing.

```
Producer → Task Queue → Worker 1
                     ↘ Worker 2
                     ↘ Worker 3
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Define task
public class ImageProcessingTask : Command
{
    public string ImageId { get; set; }
    public string[] Operations { get; set; }
}

// Producer
foreach (var image in images)
{
    await publisher.PublishCommandAsync(
        new ImageProcessingTask
        {
            ImageId = image.Id,
            Operations = new[] { 
                "resize:800x600", 
                "compress:85" 
            }
        });
}

// Worker pool
services.AddMmateWorkerPool<ImageWorker>(
    options =>
    {
        options.WorkerCount = 10;
        options.QueueName = "image.tasks";
        options.PrefetchCount = 2;
    });

// Worker implementation
public class ImageWorker : 
    IMessageHandler<ImageProcessingTask>
{
    public async Task HandleAsync(
        ImageProcessingTask task)
    {
        var image = await storage
            .LoadImage(task.ImageId);
        
        foreach (var op in task.Operations)
        {
            image = await ProcessOperation(
                image, op);
        }
        
        await storage.SaveImage(image);
    }
}
```

</td>
<td>

```go
// Define task
type ImageProcessingTask struct {
    contracts.BaseCommand
    ImageID    string   `json:"imageId"`
    Operations []string `json:"operations"`
}

// Producer
for _, image := range images {
    err := publisher.PublishCommand(ctx,
        &ImageProcessingTask{
            BaseCommand: contracts.NewBaseCommand(
                "ImageProcessingTask"),
            ImageID:    image.ID,
            Operations: []string{
                "resize:800x600",
                "compress:85",
            },
        })
}

// Worker pool
pool := messaging.NewConsumerGroup(
    "image-workers",
    messaging.WithGroupSize(10),
    messaging.WithPrefetchCount(2),
)

// Worker implementation
type ImageWorker struct {
    storage ImageStorage
}

func (w *ImageWorker) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    task := msg.(*ImageProcessingTask)
    image, err := w.storage.LoadImage(
        ctx, task.ImageID)
    if err != nil {
        return err
    }
    
    for _, op := range task.Operations {
        image, err = w.processOperation(
            image, op)
        if err != nil {
            return err
        }
    }
    
    return w.storage.SaveImage(ctx, image)
}

// Start workers
err := pool.Subscribe(ctx,
    "image.tasks",
    &ImageWorker{storage: storage})
```

</td>
</tr>
</table>

### When to Use

✅ **Good for:**
- CPU/IO intensive tasks
- Batch processing
- Rate-limited operations
- Scalable processing

❌ **Not for:**
- Ordered processing
- Real-time requirements
- Tasks needing coordination

## Topic Routing

### Overview

Route messages based on routing patterns, allowing selective message consumption.

```
Publisher → Topic Exchange → Queue 1 (*.critical)
                          ↘ Queue 2 (orders.*)
                          ↘ Queue 3 (*.*.shipped)
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Publish with routing key
await publisher.PublishEventAsync(
    new OrderEvent 
    { 
        OrderId = "123",
        Status = "shipped",
        Priority = "high"
    },
    options => options
        .WithRoutingKey("orders.high.shipped"));

// Subscribe with patterns
// All high priority orders
await subscriber.SubscribeAsync<OrderEvent>(
    "high-priority-orders",
    handler,
    options => options
        .WithTopicPattern("*.high.*"));

// All shipped orders
await subscriber.SubscribeAsync<OrderEvent>(
    "shipped-orders",
    handler,
    options => options
        .WithTopicPattern("*.*.shipped"));

// Orders from specific region
await subscriber.SubscribeAsync<OrderEvent>(
    "us-west-orders",
    handler,
    options => options
        .WithTopicPattern("orders.*.us-west"));
```

</td>
<td>

```go
// Publish with routing key
err := publisher.PublishEvent(ctx,
    &OrderEvent{
        OrderID:  "123",
        Status:   "shipped", 
        Priority: "high",
    },
    messaging.WithRoutingKey(
        "orders.high.shipped"))

// Subscribe with patterns
// All high priority orders
err := subscriber.Subscribe(ctx,
    "high-priority-orders",
    "OrderEvent",
    handler,
    messaging.WithBindingKey("*.high.*"))

// All shipped orders  
err := subscriber.Subscribe(ctx,
    "shipped-orders",
    "OrderEvent",
    handler,
    messaging.WithBindingKey("*.*.shipped"))

// Orders from specific region
err := subscriber.Subscribe(ctx,
    "us-west-orders", 
    "OrderEvent",
    handler,
    messaging.WithBindingKey(
        "orders.*.us-west"))
```

</td>
</tr>
</table>

### Routing Key Patterns

- `*` matches exactly one word
- `#` matches zero or more words
- `.` separates words

Examples:
- `orders.*.created` matches `orders.electronics.created`
- `*.critical.*` matches `payments.critical.failed`
- `logs.#` matches `logs.app.error.db`

## Saga/Workflow

### Overview

Orchestrate multi-step processes with compensation support using StageFlow.

```
Start → Stage 1 → Stage 2 → Stage 3 → Complete
         ↓ fail     ↓ fail     ↓ fail
      Compensate ← Compensate ← Compensate
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Define workflow context
public class OrderContext : WorkflowContext
{
    public string OrderId { get; set; }
    public List<OrderItem> Items { get; set; }
    public PaymentInfo Payment { get; set; }
    public ShippingInfo Shipping { get; set; }
    
    // State
    public bool InventoryReserved { get; set; }
    public string PaymentId { get; set; }
    public string TrackingNumber { get; set; }
}

// Define workflow
public class OrderWorkflow : 
    Workflow<OrderContext>
{
    protected override void Configure(
        IWorkflowBuilder<OrderContext> builder)
    {
        builder
            .AddStage<ValidateOrderStage>()
            .AddStage<ReserveInventoryStage>()
                .WithCompensation<ReleaseInventoryStage>()
            .AddStage<ProcessPaymentStage>()
                .WithCompensation<RefundPaymentStage>()
            .AddStage<CreateShipmentStage>()
            .AddStage<SendNotificationStage>();
    }
}

// Stage implementation
public class ProcessPaymentStage : 
    IWorkflowStage<OrderContext>
{
    public async Task ExecuteAsync(
        OrderContext context)
    {
        var result = await paymentService
            .ProcessPayment(new PaymentRequest
            {
                Amount = context.CalculateTotal(),
                Method = context.Payment.Method,
                CustomerId = context.Payment.CustomerId
            });
            
        context.PaymentId = result.TransactionId;
        context.PaymentProcessed = true;
    }
}

// Execute workflow
var context = new OrderContext 
{ 
    OrderId = orderId,
    Items = items 
};

var result = await workflow
    .ExecuteAsync(context);
```

</td>
<td>

```go
// Define workflow context  
type OrderContext struct {
    OrderID    string      `json:"orderId"`
    Items      []OrderItem `json:"items"`
    Payment    PaymentInfo `json:"payment"`
    Shipping   ShippingInfo `json:"shipping"`
    
    // State
    InventoryReserved bool   `json:"inventoryReserved"`
    PaymentID        string `json:"paymentId"`
    TrackingNumber   string `json:"trackingNumber"`
}

// Create workflow
workflow := stageflow.NewFlow[*OrderContext](
    "order-processing",
    stageflow.WithTimeout(10*time.Minute))

// Add stages with compensation
workflow.AddStage("validate",
    &ValidateOrderStage{})

workflow.AddStage("reserve-inventory",
    &ReserveInventoryStage{inv: inventory})
workflow.AddCompensation("reserve-inventory",
    &ReleaseInventoryStage{inv: inventory})

workflow.AddStage("process-payment",
    &ProcessPaymentStage{pay: payment})
workflow.AddCompensation("process-payment",
    &RefundPaymentStage{pay: payment})

workflow.AddStage("create-shipment",
    &CreateShipmentStage{ship: shipping})

workflow.AddStage("send-notification",
    &SendNotificationStage{notify: notifier})

// Stage implementation
type ProcessPaymentStage struct {
    pay PaymentService
}

func (s *ProcessPaymentStage) Execute(
    ctx context.Context,
    wfCtx *OrderContext) error {
    
    result, err := s.pay.ProcessPayment(ctx,
        PaymentRequest{
            Amount:     wfCtx.CalculateTotal(),
            Method:     wfCtx.Payment.Method,
            CustomerID: wfCtx.Payment.CustomerID,
        })
    if err != nil {
        return err
    }
    
    wfCtx.PaymentID = result.TransactionID
    wfCtx.PaymentProcessed = true
    return nil
}

// Execute workflow
context := &OrderContext{
    OrderID: orderID,
    Items:   items,
}

result, err := workflow.Execute(ctx, context)
```

</td>
</tr>
</table>

### When to Use

✅ **Good for:**
- Order processing
- Multi-service transactions  
- Complex business processes
- Processes needing rollback

❌ **Not for:**
- Simple operations
- High-frequency operations
- Real-time requirements

## Pattern Selection Guide

| If you need to... | Use this pattern |
|-------------------|------------------|
| Notify multiple services of an event | Publish/Subscribe |
| Get a response to a request | Request/Reply |
| Distribute work among instances | Work Queue |
| Route messages by criteria | Topic Routing |
| Orchestrate multi-step process | Saga/Workflow |
| Process messages in order | Work Queue with single consumer |
| Handle high throughput | Work Queue with many consumers |
| Implement event sourcing | Publish/Subscribe with persistence |

## Combining Patterns

Patterns can be combined for complex scenarios:

### Event-Driven Saga
1. Event triggers workflow start
2. Workflow orchestrates multiple services
3. Each stage may publish events
4. Compensation handles failures

### Request/Reply with Timeout and Retry
1. Bridge sends request
2. Retry interceptor handles transient failures
3. Circuit breaker prevents cascading failures
4. Timeout ensures bounded wait time

### Work Queue with Priority
1. Multiple priority queues
2. Workers consume high-priority first
3. Topic routing directs to correct queue
4. Dead letter queue handles failures

## Best Practices

### General
- Choose the simplest pattern that meets your needs
- Design for failure - use timeouts, retries, and circuit breakers
- Monitor queue depths and processing times
- Use correlation IDs for tracing

### Pattern-Specific

**Publish/Subscribe**
- Make events immutable
- Include all necessary data
- Version events for compatibility

**Request/Reply**
- Set appropriate timeouts
- Handle timeout scenarios
- Consider async alternatives

**Work Queue**
- Make tasks idempotent
- Size prefetch appropriately
- Monitor worker utilization

**Topic Routing**
- Design routing key hierarchy carefully
- Document routing patterns
- Avoid overly complex patterns

**Saga/Workflow**
- Keep stages focused
- Always implement compensation
- Persist state between stages

## Next Steps

- Explore [Component Documentation](../components/README.md)
- Review [Platform Examples](../README.md#examples)
- Learn about [Reliability Patterns](../advanced/reliability.md)
- Study [Real-World Scenarios](../advanced/scenarios.md)