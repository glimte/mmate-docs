# Mmate Components

This section documents the core components that make up the Mmate messaging framework. Each component is designed to work seamlessly together while maintaining clear separation of concerns.

## Component Overview

| Component | Purpose | Key Features |
|-----------|---------|--------------|
| [Contracts](contracts/README.md) | Message type definitions | Commands, Events, Queries, Replies |
| [Messaging](messaging/README.md) | Core pub/sub functionality | Publishers, Subscribers, Handlers |
| [StageFlow](stageflow.md) | Workflow orchestration | Multi-stage processes, Queue-based Compensation |
| [Bridge](bridge/README.md) | Sync-over-async patterns | Request/Reply, Timeouts |
| [Interceptors](interceptors/README.md) | Cross-cutting concerns | Logging, Metrics, Validation |
| [Monitoring](monitoring/README.md) | Observability | Health checks, Metrics, Dashboards |

## Component Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Code                          │
├─────────────────────────────────────────────────────────────┤
│  StageFlow  │  Bridge  │        User Handlers               │
├─────────────┬─────────┬─────────────────────────────────────┤
│             │         │        Interceptors                  │
│             ├─────────┴─────────────────────────────────────┤
│             │            Messaging Core                      │
│             │   (Publisher, Subscriber, Dispatcher)          │
├─────────────┴───────────────────────────────────────────────┤
│                      Contracts                               │
│              (Message Types & Interfaces)                    │
├─────────────────────────────────────────────────────────────┤
│                 Transport (RabbitMQ)                         │
└─────────────────────────────────────────────────────────────┘
                              ↕
                         Monitoring
```

## Quick Start

### 1. Define Your Messages (Contracts)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class CreateOrderCommand : Command
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderCreatedEvent : Event
{
    public string OrderId { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

</td>
<td>

```go
type CreateOrderCommand struct {
    contracts.BaseCommand
    CustomerID string      `json:"customerId"`
    Items      []OrderItem `json:"items"`
}

type OrderCreatedEvent struct {
    contracts.BaseEvent
    OrderID   string    `json:"orderId"`
    CreatedAt time.Time `json:"createdAt"`
}
```

</td>
</tr>
</table>

### 2. Publish Messages (Messaging)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
await publisher.PublishCommandAsync(
    new CreateOrderCommand
    {
        CustomerId = "CUST-123",
        Items = orderItems
    });
```

</td>
<td>

```go
err := publisher.PublishCommand(ctx,
    &CreateOrderCommand{
        CustomerID: "CUST-123",
        Items:     orderItems,
    })
```

</td>
</tr>
</table>

### 3. Handle Messages (Messaging)

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
        CreateOrderCommand cmd)
    {
        // Process order
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
    // Process order
    return nil
}
```

</td>
</tr>
</table>

### 4. Add Cross-Cutting Concerns (Interceptors)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmate(options =>
{
    options.AddInterceptor<LoggingInterceptor>();
    options.AddInterceptor<MetricsInterceptor>();
});
```

</td>
<td>

```go
pipeline := interceptors.NewPipeline(
    &LoggingInterceptor{},
    &MetricsInterceptor{},
)
```

</td>
</tr>
</table>

### 5. Orchestrate Workflows (StageFlow)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderWorkflow : 
    Workflow<OrderContext>
{
    protected override void Configure(
        IWorkflowBuilder<OrderContext> builder)
    {
        builder
            .AddStage<ValidateOrder>()
            .AddStage<ProcessPayment>()
                .WithCompensation<RefundPayment>()
            .AddStage<ShipOrder>()
                .WithCompensation<CancelShipment>();
    }
}
```

</td>
<td>

```go
workflow := stageflow.NewTypedWorkflow[*OrderContext](
    "order-processing", "Order Processing")

workflow.AddTypedStage("validate", &ValidateOrder{})
workflow.AddTypedStage("payment", &ProcessPayment{}).
    WithCompensation(&RefundPayment{})
workflow.AddTypedStage("shipping", &ShipOrder{}).
    WithCompensation(&CancelShipment{})

// Queue-based compensation handled automatically
workflow.Build()
```

</td>
</tr>
</table>

### 6. Enable Request/Reply (Bridge)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
var reply = await bridge
    .SendAndWaitAsync<PriceReply>(
        new GetPriceQuery 
        { 
            ProductId = "PROD-123" 
        });
```

</td>
<td>

```go
reply, err := bridge.SendAndWait(ctx,
    &GetPriceQuery{
        ProductID: "PROD-123",
    },
    "qry.pricing.get",
    10*time.Second)
```

</td>
</tr>
</table>

### 7. Monitor Your System (Monitoring)

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddHealthChecks()
    .AddRabbitMQ()
    .AddCheck<MessageProcessingCheck>(
        "processing");

app.MapHealthChecks("/health");
```

</td>
<td>

```go
health := monitoring.NewHealthChecker()
http.HandleFunc("/health", 
    health.ServeHTTP)
```

</td>
</tr>
</table>

## Component Selection Guide

| If you need to... | Use this component |
|-------------------|-------------------|
| Define message structures | [Contracts](contracts/README.md) |
| Send/receive messages | [Messaging](messaging/README.md) |
| Get responses to requests | [Bridge](bridge/README.md) |
| Orchestrate multi-step processes | [StageFlow](stageflow/README.md) |
| Add logging/metrics/auth | [Interceptors](interceptors/README.md) |
| Monitor system health | [Monitoring](monitoring/README.md) |

## Integration Patterns

### 1. Basic Messaging
Contracts → Messaging → Handlers

### 2. Request/Response
Contracts → Bridge → Messaging → Handlers

### 3. Workflow Processing
Contracts → StageFlow → Messaging → Handlers

### 4. Full Stack
All components working together with interceptors and monitoring

## Best Practices

1. **Start Simple**
   - Begin with basic messaging
   - Add components as needed
   - Don't over-engineer

2. **Use Contracts**
   - Define clear message contracts
   - Version your messages
   - Document message flows

3. **Apply Interceptors**
   - Use for cross-cutting concerns
   - Keep them lightweight
   - Order matters

4. **Monitor Everything**
   - Set up health checks early
   - Track key metrics
   - Configure alerts

5. **Test Components**
   - Unit test handlers
   - Integration test workflows
   - Load test critical paths

## Next Steps

- Dive into specific [component documentation](contracts/README.md)
- Review [examples](../README.md#examples)
- Learn about [patterns](../patterns/README.md)
- Explore [advanced topics](../advanced/README.md)