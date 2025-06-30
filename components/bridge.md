# Bridge / SyncAsyncBridge - Synchronous Over Asynchronous

The Bridge component (Go) / SyncAsyncBridge (.NET) enables synchronous request-response patterns over asynchronous messaging infrastructure, providing RPC-style communication with the reliability of message queuing.

## Overview

The Bridge pattern allows you to:
- Make request-response calls that feel synchronous
- Wait for responses with configurable timeouts
- Correlate requests and responses automatically
- Handle both commands and queries uniformly
- Maintain location transparency

## Core Concepts

### Request-Response Pattern
```
Client → Bridge → Request Queue → Handler
Client ← Bridge ← Reply Queue   ← Handler
```

### Correlation
Each request is assigned a unique correlation ID that links it to its response, enabling multiple concurrent requests.

### Timeout Handling
Requests have configurable timeouts to prevent indefinite waiting and provide predictable behavior.

## Basic Usage

### Making Requests

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Create bridge
var bridge = new SyncAsyncBridge(
    publisher, subscriber, logger);

// Send command and wait for reply
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    new CreateOrderCommand
    {
        CustomerId = "CUST-123",
        Items = orderItems
    },
    timeout: TimeSpan.FromSeconds(30));

if (reply.Success)
{
    Console.WriteLine($"Order created: {reply.OrderId}");
}
else
{
    Console.WriteLine($"Failed: {reply.Error}");
}

// Send query
var result = await bridge.SendAndWaitAsync<PriceReply>(
    new GetPriceQuery
    {
        ProductId = "PROD-456",
        Quantity = 5
    });

Console.WriteLine($"Total price: {result.TotalPrice}");
```

</td>
<td>

```go
// Create bridge
bridgeClient := bridge.NewSyncAsyncBridge(
    publisher, subscriber,
    bridge.WithReplyQueue("my-service-reply"))

// Send command and wait for reply
reply, err := bridgeClient.RequestCommand(ctx,
    &CreateOrderCommand{
        BaseCommand: contracts.BaseCommand{
            BaseMessage: contracts.BaseMessage{
                Type: "CreateOrderCommand",
                ID:   uuid.New().String(),
                Timestamp: time.Now(),
            },
            TargetService: "order-service",
        },
        CustomerID: "CUST-123",
        Items:     orderItems,
    },
    30*time.Second)

if err != nil {
    return err
}

// Type assertion required
orderReply, ok := reply.(*OrderReply)
if !ok {
    return fmt.Errorf("unexpected reply type")
}

if orderReply.Success {
    fmt.Printf("Order created: %s\n", orderReply.OrderID)
} else {
    fmt.Printf("Failed: %s\n", orderReply.Error)
}

// Send query
result, err := bridgeClient.RequestQuery(ctx,
    &GetPriceQuery{
        BaseQuery: contracts.BaseQuery{
            BaseMessage: contracts.BaseMessage{
                Type: "GetPriceQuery",
                ID:   uuid.New().String(),
                Timestamp: time.Now(),
            },
        },
        ProductID: "PROD-456",
        Quantity:  5,
    },
    10*time.Second)

if err != nil {
    return err
}

priceReply := result.(*PriceReply)
fmt.Printf("Total price: %.2f\n", priceReply.TotalPrice)

// NEW: Type-safe helpers (available since v0.2.0)
// Eliminates type assertions
orderReply, err := bridge.RequestCommandTyped[*OrderReply](
    bridgeClient, ctx, cmd, 30*time.Second)
// No type assertion needed!

priceReply, err := bridge.RequestQueryTyped[*PriceReply](
    bridgeClient, ctx, query, 10*time.Second)
// Direct typed access
```

</td>
</tr>
</table>

### Handling Requests

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Command handler with reply
public class CreateOrderHandler : 
    IRequestHandler<CreateOrderCommand, OrderReply>
{
    private readonly IOrderService _orderService;
    
    public async Task<OrderReply> HandleAsync(
        CreateOrderCommand command,
        RequestContext context)
    {
        try
        {
            var orderId = await _orderService
                .CreateOrder(command);
            
            return new OrderReply
            {
                Success = true,
                OrderId = orderId,
                Message = "Order created successfully"
            };
        }
        catch (ValidationException ex)
        {
            return new OrderReply
            {
                Success = false,
                Error = ex.Message,
                ValidationErrors = ex.Errors
            };
        }
    }
}

// Query handler
public class GetPriceHandler : 
    IRequestHandler<GetPriceQuery, PriceReply>
{
    public async Task<PriceReply> HandleAsync(
        GetPriceQuery query,
        RequestContext context)
    {
        var price = await _catalog
            .GetPrice(query.ProductId);
        
        var total = price * query.Quantity;
        var discount = CalculateDiscount(
            query.Quantity);
        
        return new PriceReply
        {
            UnitPrice = price,
            Quantity = query.Quantity,
            TotalPrice = total,
            Discount = discount,
            FinalPrice = total - discount
        };
    }
}
```

</td>
<td>

```go
// Command handler with reply
type CreateOrderHandler struct {
    orderService OrderService
    publisher    messaging.Publisher
}

func (h *CreateOrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CreateOrderCommand)
    
    orderID, err := h.orderService.CreateOrder(
        ctx, cmd)
    
    var reply *OrderReply
    if err != nil {
        reply = &OrderReply{
            BaseReply: contracts.BaseReply{
                BaseMessage: contracts.BaseMessage{
                    Type:          "OrderReply",
                    ID:            uuid.New().String(),
                    Timestamp:     time.Now(),
                    CorrelationID: cmd.GetCorrelationID(),
                },
                Success: false,
            },
            Error: err.Error(),
        }
        
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            reply.ValidationErrors = validationErr.Errors
        }
    } else {
        reply = &OrderReply{
            BaseReply: contracts.BaseReply{
                BaseMessage: contracts.BaseMessage{
                    Type:          "OrderReply",
                    ID:            uuid.New().String(),
                    Timestamp:     time.Now(),
                    CorrelationID: cmd.GetCorrelationID(),
                },
                Success: true,
            },
            OrderID: orderID,
            Message: "Order created successfully",
        }
    }
    
    // IMPORTANT: Must use PublishReply for direct queue routing
    return h.publisher.PublishReply(
        ctx, reply, cmd.ReplyTo)
}

// Query handler
type GetPriceHandler struct {
    catalog   PriceCatalog
    publisher messaging.Publisher
}

func (h *GetPriceHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    query := msg.(*GetPriceQuery)
    
    price, err := h.catalog.GetPrice(
        ctx, query.ProductID)
    if err != nil {
        return err
    }
    
    total := price * float64(query.Quantity)
    discount := h.calculateDiscount(query.Quantity)
    
    reply := &PriceReply{
        BaseReply: contracts.BaseReply{
            BaseMessage: contracts.BaseMessage{
                Type:          "PriceReply",
                ID:            uuid.New().String(),
                Timestamp:     time.Now(),
                CorrelationID: query.GetCorrelationID(),
            },
        },
        UnitPrice:  price,
        Quantity:   query.Quantity,
        TotalPrice: total,
        Discount:   discount,
        FinalPrice: total - discount,
    }
    
    // IMPORTANT: Must use PublishReply for direct queue routing
    return h.publisher.PublishReply(
        ctx, reply, query.ReplyTo)
}
```

</td>
</tr>
</table>

## Advanced Features

### Type-Safe Bridge Operations (Go)

Since v0.2.0, Go provides type-safe helper functions that eliminate runtime type assertions:

```go
// Traditional approach with type assertion
reply, err := bridge.RequestCommand(ctx, cmd, timeout)
orderReply, ok := reply.(*OrderReply)
if !ok {
    return fmt.Errorf("unexpected reply type")
}

// New type-safe approach
orderReply, err := bridge.RequestCommandTyped[*OrderReply](
    bridge, ctx, cmd, timeout)
// Direct typed access - no assertion needed!

// Works for queries too
priceReply, err := bridge.RequestQueryTyped[*PriceReply](
    bridge, ctx, query, timeout)
```

### Configuration Options

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure bridge
var bridge = new SyncAsyncBridge(
    publisher, subscriber, logger,
    options =>
    {
        options.ReplyQueue = "my-service-reply";
        options.CleanupInterval = TimeSpan.FromMinutes(1);
        options.MaxPendingRequests = 1000;
        options.DefaultTimeout = TimeSpan.FromSeconds(30);
    });
```

</td>
<td>

```go
// Configure bridge
bridge := bridge.NewSyncAsyncBridge(
    publisher, subscriber,
    bridge.WithReplyQueue("my-service-reply"),
    bridge.WithCleanupInterval(time.Minute),
    bridge.WithMaxPendingRequests(1000),
    bridge.WithDefaultTimeout(30*time.Second),
    bridge.WithBridgeLogger(logger))

// Circuit breaker (configured at client level)
circuitBreaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(time.Minute),
    reliability.WithSuccessThreshold(2))

// Retry policy (configured at client level)
retryPolicy := reliability.NewExponentialBackoff(
    time.Second,      // initial delay
    10*time.Second,   // max delay
    2.0,              // multiplier
    3,                // max attempts
)

bridge := bridge.NewSyncAsyncBridge(
    publisher, subscriber,
    bridge.WithBridgeCircuitBreaker(circuitBreaker),
    bridge.WithBridgeRetryPolicy(retryPolicy))
```

</td>
</tr>
</table>

### Error Handling

#### Timeout Handling

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
try
{
    var reply = await bridge.SendAndWaitAsync<OrderReply>(
        command,
        timeout: TimeSpan.FromSeconds(30));
}
catch (TimeoutException ex)
{
    logger.LogError(ex,
        "Request timed out after {Timeout}s",
        30);
    
    // Handle timeout - maybe try alternative action
    await fallbackService.ProcessOrder(command);
}
catch (BridgeException ex)
{
    logger.LogError(ex,
        "Bridge error: {Error}",
        ex.Message);
}
```

</td>
<td>

```go
reply, err := bridge.RequestCommand(ctx,
    command, 30*time.Second)

if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        logger.Error("Request timed out",
            "timeout", "30s")
        
        // Handle timeout - maybe try alternative
        return fallbackService.ProcessOrder(ctx, command)
    }
    
    return fmt.Errorf("bridge request failed: %w", err)
}
```

</td>
</tr>
</table>

## Implementation Details

### Go: Key Differences from Documentation

1. **No routing key parameter**: The actual implementation doesn't use routing keys in `RequestCommand`/`RequestQuery`
2. **ReplyTo field is set automatically**: Using reflection to set the ReplyTo field on commands/queries
3. **Must use PublishReply**: Handlers must use `publisher.PublishReply(ctx, reply, cmd.ReplyTo)` with empty exchange for direct queue routing
4. **Type-safe helpers are functions**: Due to Go's lack of generic methods, type-safe helpers are package-level functions

### Message Structure Requirements

Commands and Queries must embed the appropriate base types:

```go
// Command must embed BaseCommand
type MyCommand struct {
    contracts.BaseCommand
    // Your fields...
}

// Query must embed BaseQuery  
type MyQuery struct {
    contracts.BaseQuery
    // Your fields...
}

// Reply must embed BaseReply
type MyReply struct {
    contracts.BaseReply
    Success bool   `json:"success"`
    Error   string `json:"error,omitempty"`
    // Your fields...
}
```

## Monitoring

### Metrics

Key metrics to track:
- Request rate
- Response time (p50, p95, p99)
- Timeout rate
- Error rate
- Pending request count
- Reply queue depth

### Health Checks

```go
// Check bridge health
pendingCount := bridge.GetPendingRequestCount()
if pendingCount > 900 { // 90% of max
    logger.Warn("Bridge nearing capacity",
        "pending", pendingCount,
        "max", 1000)
}
```

## Best Practices

1. **Always set appropriate timeouts**
   - Consider network latency and processing time
   - Use shorter timeouts for queries than commands
   - Set timeouts based on SLAs

2. **Handle all reply types**
   - Always check Success field in replies
   - Handle validation errors separately
   - Log correlation IDs for debugging

3. **Use PublishReply for responses**
   - Don't use regular Publish with routing keys
   - Ensure empty exchange for direct queue routing
   - Always include correlation ID

4. **Leverage type-safe helpers (Go)**
   - Use `RequestCommandTyped` and `RequestQueryTyped`
   - Eliminates runtime type assertion errors
   - Better IDE support and compile-time safety

5. **Monitor pending requests**
   - Track queue depths
   - Set alerts for high pending counts
   - Implement circuit breakers for protection

## Common Use Cases

### API Gateway Pattern
Convert synchronous HTTP requests to async messages:
- HTTP endpoints use Bridge for backend calls
- Maintains REST-like interface
- Benefits of message queuing

### Service-to-Service Communication
Replace HTTP calls with reliable messaging:
- Better fault tolerance
- Automatic retries
- Location transparency
- Load balancing

### Legacy System Integration
Wrap async systems with synchronous interface:
- Easier migration path
- Familiar programming model
- Gradual modernization

## Next Steps

- Learn about [StageFlow](../stageflow/README.md) for workflow orchestration
- Implement [Interceptors](../interceptors/README.md) for cross-cutting concerns
- Add [Monitoring](../monitoring/README.md) to track performance
- Review [Examples](../../README.md#examples) for real-world usage