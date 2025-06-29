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
bridge := bridge.NewSyncAsyncBridge(
    publisher, subscriber, logger)

// Send command and wait for reply
reply, err := bridge.SendAndWait(ctx,
    &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand(
            "CreateOrderCommand"),
        CustomerID: "CUST-123",
        Items:     orderItems,
    },
    "cmd.orders.create",
    30*time.Second)

if err != nil {
    return err
}

orderReply := reply.(*OrderReply)
if orderReply.Success {
    fmt.Printf("Order created: %s\n", 
        orderReply.OrderID)
} else {
    fmt.Printf("Failed: %s\n", 
        orderReply.Error)
}

// Send query
result, err := bridge.SendAndWait(ctx,
    &GetPriceQuery{
        BaseQuery: contracts.NewBaseQuery(
            "GetPriceQuery"),
        ProductID: "PROD-456",
        Quantity:  5,
    },
    "qry.pricing.get",
    10*time.Second)

priceReply := result.(*PriceReply)
fmt.Printf("Total price: %.2f\n", 
    priceReply.TotalPrice)
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
    
    var reply contracts.Reply
    if err != nil {
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            reply = &OrderReply{
                BaseReply: contracts.NewBaseReply(
                    cmd.GetID(), 
                    cmd.GetCorrelationID()),
                Success:          false,
                Error:           err.Error(),
                ValidationErrors: validationErr.Errors,
            }
        } else {
            return err
        }
    } else {
        reply = &OrderReply{
            BaseReply: contracts.NewBaseReply(
                cmd.GetID(), 
                cmd.GetCorrelationID()),
            Success: true,
            OrderID: orderID,
            Message: "Order created successfully",
        }
    }
    
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
        BaseReply: contracts.NewBaseReply(
            query.GetID(),
            query.GetCorrelationID()),
        UnitPrice:  price,
        Quantity:   query.Quantity,
        TotalPrice: total,
        Discount:   discount,
        FinalPrice: total - discount,
    }
    
    return h.publisher.PublishReply(
        ctx, reply, query.ReplyTo)
}
```

</td>
</tr>
</table>

## Advanced Features

### Batch Operations

Process multiple requests concurrently:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Batch requests
var requests = products.Select(p => 
    new GetPriceQuery 
    { 
        ProductId = p.Id,
        Quantity = p.Quantity
    }).ToList();

// Send all requests concurrently
var tasks = requests.Select(req =>
    bridge.SendAndWaitAsync<PriceReply>(
        req, TimeSpan.FromSeconds(10)));

// Wait for all responses
var replies = await Task.WhenAll(tasks);

// Process results
var totalAmount = replies
    .Where(r => r.Success)
    .Sum(r => r.FinalPrice);

// With cancellation
using var cts = new CancellationTokenSource(
    TimeSpan.FromSeconds(30));

var replies = await bridge
    .SendBatchAsync<GetPriceQuery, PriceReply>(
        requests,
        timeout: TimeSpan.FromSeconds(10),
        cancellationToken: cts.Token);
```

</td>
<td>

```go
// Batch requests
var requests []contracts.Message
for _, p := range products {
    requests = append(requests, &GetPriceQuery{
        BaseQuery: contracts.NewBaseQuery(
            "GetPriceQuery"),
        ProductID: p.ID,
        Quantity:  p.Quantity,
    })
}

// Send all requests concurrently
g, ctx := errgroup.WithContext(ctx)
replies := make([]contracts.Reply, len(requests))

for i, req := range requests {
    i, req := i, req // capture
    g.Go(func() error {
        reply, err := bridge.SendAndWait(
            ctx, req, "qry.pricing.get",
            10*time.Second)
        if err != nil {
            return err
        }
        replies[i] = reply
        return nil
    })
}

// Wait for all responses
if err := g.Wait(); err != nil {
    return err
}

// Process results
var totalAmount float64
for _, r := range replies {
    if priceReply, ok := r.(*PriceReply); ok {
        if priceReply.Success {
            totalAmount += priceReply.FinalPrice
        }
    }
}

// Note: Batch helper is not currently implemented
// Use goroutines and errgroup for concurrent requests as shown above
```

</td>
</tr>
</table>

### Retry and Circuit Breaking

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure retry policy
var bridge = new SyncAsyncBridge(
    publisher, subscriber, logger,
    options =>
    {
        options.RetryPolicy = new ExponentialBackoffRetry
        {
            MaxAttempts = 3,
            InitialDelay = TimeSpan.FromSeconds(1),
            MaxDelay = TimeSpan.FromSeconds(10)
        };
        
        options.CircuitBreaker = new CircuitBreakerOptions
        {
            FailureThreshold = 5,
            ResetTimeout = TimeSpan.FromMinutes(1),
            HalfOpenAttempts = 3
        };
    });

// Use with specific retry policy
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    command,
    options => options
        .WithTimeout(TimeSpan.FromSeconds(30))
        .WithRetryPolicy(new FixedDelayRetry
        {
            MaxAttempts = 5,
            Delay = TimeSpan.FromSeconds(2)
        })
        .OnRetry((attempt, delay) =>
        {
            logger.LogWarning(
                "Retry {Attempt} after {Delay}ms",
                attempt, delay.TotalMilliseconds);
        }));
```

</td>
<td>

```go
// Retry and circuit breaker are configured at client level
retryPolicy := reliability.NewExponentialBackoff(
    time.Second,      // initial delay
    10*time.Second,   // max delay
    2.0,              // multiplier
    3,                // max attempts
)

circuitBreaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(time.Minute),
    reliability.WithSuccessThreshold(2))

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithRetryPolicy(retryPolicy),
    mmate.WithCircuitBreaker(circuitBreaker))

// Bridge is accessed via client
reply, err := client.Bridge().SendAndWait(ctx,
    command,
    "cmd.orders.create",
    30*time.Second)
```

</td>
</tr>
</table>

### Streaming Responses

Handle large result sets with streaming:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Stream results
public async IAsyncEnumerable<OrderDto> 
    SearchOrdersAsync(
        SearchOrdersQuery query,
        [EnumeratorCancellation] 
        CancellationToken ct = default)
{
    var pageSize = 100;
    var page = 1;
    
    while (!ct.IsCancellationRequested)
    {
        var pagedQuery = query with
        {
            Page = page,
            PageSize = pageSize
        };
        
        var reply = await bridge
            .SendAndWaitAsync<SearchOrdersReply>(
                pagedQuery,
                timeout: TimeSpan.FromSeconds(30),
                cancellationToken: ct);
        
        foreach (var order in reply.Orders)
        {
            yield return order;
        }
        
        if (!reply.HasMore)
            break;
            
        page++;
    }
}

// Consume stream
await foreach (var order in SearchOrdersAsync(query))
{
    await ProcessOrder(order);
}
```

</td>
<td>

```go
// Stream results
func (b *Bridge) StreamOrders(
    ctx context.Context,
    query *SearchOrdersQuery) (<-chan *OrderDto, error) {
    
    orders := make(chan *OrderDto)
    
    go func() {
        defer close(orders)
        
        pageSize := 100
        page := 1
        
        for {
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            pagedQuery := &SearchOrdersQuery{
                BaseQuery: query.BaseQuery,
                Filter:    query.Filter,
                Page:      page,
                PageSize:  pageSize,
            }
            
            reply, err := b.SendAndWait(ctx,
                pagedQuery,
                "qry.orders.search",
                30*time.Second)
            if err != nil {
                return
            }
            
            searchReply := reply.(*SearchOrdersReply)
            for _, order := range searchReply.Orders {
                select {
                case orders <- order:
                case <-ctx.Done():
                    return
                }
            }
            
            if !searchReply.HasMore {
                return
            }
            
            page++
        }
    }()
    
    return orders, nil
}

// Consume stream
orders, err := bridge.StreamOrders(ctx, query)
if err != nil {
    return err
}

for order := range orders {
    if err := processOrder(order); err != nil {
        return err
    }
}
```

</td>
</tr>
</table>

### Request Context

Pass additional context with requests:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Send with context
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    command,
    options => options
        .WithHeader("userId", currentUser.Id)
        .WithHeader("tenantId", tenant.Id)
        .WithHeader("source", "web-api")
        .WithTraceContext(Activity.Current));

// Access in handler
public class OrderHandler : 
    IRequestHandler<CreateOrderCommand, OrderReply>
{
    public async Task<OrderReply> HandleAsync(
        CreateOrderCommand command,
        RequestContext context)
    {
        var userId = context.Headers["userId"];
        var tenantId = context.Headers["tenantId"];
        var traceId = context.TraceContext?.TraceId;
        
        logger.LogInformation(
            "Processing order for user {UserId} " +
            "in tenant {TenantId}",
            userId, tenantId);
        
        // Process with context...
    }
}
```

</td>
<td>

```go
// Send with context
ctx = metadata.AppendToOutgoingContext(ctx,
    "userId", currentUser.ID,
    "tenantId", tenant.ID,
    "source", "web-api")

reply, err := bridge.SendAndWait(ctx,
    command,
    "cmd.orders.create",
    30*time.Second,
    bridge.WithHeaders(map[string]string{
        "userId":   currentUser.ID,
        "tenantId": tenant.ID,
        "source":   "web-api",
    }))

// Access in handler
func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CreateOrderCommand)
    
    userID := msg.GetHeader("userId")
    tenantID := msg.GetHeader("tenantId")
    
    span := trace.SpanFromContext(ctx)
    traceID := span.SpanContext().TraceID()
    
    h.logger.Info("Processing order",
        "userId", userID,
        "tenantId", tenantID,
        "traceId", traceID)
    
    // Process with context...
}
```

</td>
</tr>
</table>

## Error Handling

### Timeout Handling

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

// With custom timeout handling
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    command,
    options => options
        .WithTimeout(TimeSpan.FromSeconds(30))
        .OnTimeout(async () =>
        {
            // Log timeout
            await metrics.RecordTimeout("CreateOrder");
            
            // Try fallback
            return new OrderReply
            {
                Success = false,
                Error = "Service temporarily unavailable"
            };
        }));
```

</td>
<td>

```go
reply, err := bridge.SendAndWait(ctx,
    command,
    "cmd.orders.create",
    30*time.Second)

if err != nil {
    var timeoutErr *bridge.TimeoutError
    if errors.As(err, &timeoutErr) {
        logger.Error("Request timed out",
            "timeout", timeoutErr.Timeout,
            "correlationId", timeoutErr.CorrelationID)
        
        // Handle timeout - maybe try alternative
        return fallbackService.ProcessOrder(ctx, command)
    }
    
    var bridgeErr *bridge.BridgeError
    if errors.As(err, &bridgeErr) {
        logger.Error("Bridge error",
            "error", bridgeErr.Error())
    }
    
    return err
}

// With custom timeout handling
reply, err := bridge.SendAndWait(ctx,
    command,
    "cmd.orders.create", 
    30*time.Second,
    bridge.WithTimeoutHandler(
        func(ctx context.Context) (contracts.Reply, error) {
            // Log timeout
            metrics.RecordTimeout("CreateOrder")
            
            // Return fallback response
            return &OrderReply{
                BaseReply: contracts.NewBaseReply(
                    command.GetID(),
                    command.GetCorrelationID()),
                Success: false,
                Error:   "Service temporarily unavailable",
            }, nil
        }))
```

</td>
</tr>
</table>

### Error Propagation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Handler with error details
public class OrderHandler : 
    IRequestHandler<CreateOrderCommand, OrderReply>
{
    public async Task<OrderReply> HandleAsync(
        CreateOrderCommand command,
        RequestContext context)
    {
        try
        {
            var orderId = await orderService
                .CreateOrder(command);
            return OrderReply.Success(orderId);
        }
        catch (ValidationException ex)
        {
            return OrderReply.ValidationError(
                ex.Message,
                ex.Errors);
        }
        catch (BusinessException ex)
        {
            return OrderReply.BusinessError(
                ex.Code,
                ex.Message);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unexpected error");
            return OrderReply.SystemError(
                "An error occurred processing your request");
        }
    }
}

// Client error handling
var reply = await bridge.SendAndWaitAsync<OrderReply>(
    command);

switch (reply.ErrorType)
{
    case ErrorType.Validation:
        ShowValidationErrors(reply.ValidationErrors);
        break;
    case ErrorType.Business:
        ShowBusinessError(reply.ErrorCode, reply.Error);
        break;
    case ErrorType.System:
        ShowSystemError(reply.Error);
        break;
}
```

</td>
<td>

```go
// Handler with error details
func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    cmd := msg.(*CreateOrderCommand)
    
    orderID, err := h.orderService.CreateOrder(
        ctx, cmd)
    
    var reply *OrderReply
    if err != nil {
        baseReply := contracts.NewBaseReply(
            cmd.GetID(), cmd.GetCorrelationID())
        
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            reply = &OrderReply{
                BaseReply:        baseReply,
                Success:          false,
                ErrorType:        ErrorTypeValidation,
                Error:           err.Error(),
                ValidationErrors: validationErr.Errors,
            }
        } else if businessErr, ok := err.(*BusinessError); ok {
            reply = &OrderReply{
                BaseReply:  baseReply,
                Success:    false,
                ErrorType:  ErrorTypeBusiness,
                ErrorCode:  businessErr.Code,
                Error:     businessErr.Message,
            }
        } else {
            h.logger.Error("Unexpected error",
                "error", err)
            reply = &OrderReply{
                BaseReply: baseReply,
                Success:   false,
                ErrorType: ErrorTypeSystem,
                Error:    "An error occurred",
            }
        }
    } else {
        reply = NewOrderSuccessReply(
            cmd.GetID(), cmd.GetCorrelationID(), orderID)
    }
    
    return h.publisher.Publish(ctx, reply,
        messaging.WithRoutingKey(cmd.ReplyTo))
}

// Client error handling
reply, err := bridge.SendAndWait(ctx,
    command, "cmd.orders.create", 30*time.Second)
if err != nil {
    return err
}

orderReply := reply.(*OrderReply)
switch orderReply.ErrorType {
case ErrorTypeValidation:
    showValidationErrors(orderReply.ValidationErrors)
case ErrorTypeBusiness:
    showBusinessError(orderReply.ErrorCode, orderReply.Error)
case ErrorTypeSystem:
    showSystemError(orderReply.Error)
}
```

</td>
</tr>
</table>

## Performance Optimization

### Connection Pooling

Reuse connections for better performance:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure connection pool
services.AddMmateBridge(options =>
{
    options.ConnectionPool = new ConnectionPoolOptions
    {
        MinConnections = 2,
        MaxConnections = 10,
        ConnectionLifetime = TimeSpan.FromMinutes(5),
        IdleTimeout = TimeSpan.FromMinutes(1)
    };
    
    options.ChannelPool = new ChannelPoolOptions
    {
        MaxChannelsPerConnection = 10,
        ChannelIdleTimeout = TimeSpan.FromSeconds(30)
    };
});
```

</td>
<td>

```go
// Configure connection pool
connManager := rabbitmq.NewConnectionManager(url,
    rabbitmq.WithMinConnections(2),
    rabbitmq.WithMaxConnections(10),
    rabbitmq.WithConnectionLifetime(5*time.Minute),
    rabbitmq.WithIdleTimeout(time.Minute))

channelPool := rabbitmq.NewChannelPool(connManager,
    rabbitmq.WithMaxChannelsPerConnection(10),
    rabbitmq.WithChannelIdleTimeout(30*time.Second))

bridge := bridge.NewSyncAsyncBridge(
    messaging.NewPublisher(channelPool),
    messaging.NewSubscriber(channelPool),
    logger)
```

</td>
</tr>
</table>

### Response Caching

Cache frequently requested data:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure caching
services.AddMmateBridge(options =>
{
    options.EnableCaching = true;
    options.CacheProvider = new RedisCacheProvider();
    options.DefaultCacheDuration = 
        TimeSpan.FromMinutes(5);
});

// Use with caching
var reply = await bridge.SendAndWaitAsync<PriceReply>(
    query,
    options => options
        .WithCaching(TimeSpan.FromMinutes(10))
        .WithCacheKey($"price:{query.ProductId}"));

// Bypass cache
var reply = await bridge.SendAndWaitAsync<PriceReply>(
    query,
    options => options.BypassCache());
```

</td>
<td>

```go
// Note: Response caching is not currently implemented
// in the Go version of the bridge.
// 
// For caching, implement at the handler level:

type CachedPriceHandler struct {
    cache    *redis.Client
    catalog  PriceCatalog
    ttl      time.Duration
}

func (h *CachedPriceHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    query := msg.(*GetPriceQuery)
    cacheKey := fmt.Sprintf("price:%s", query.ProductID)
    
    // Try cache first
    cached, err := h.cache.Get(ctx, cacheKey).Result()
    if err == nil {
        // Return cached response
        // ... deserialize and return
    }
    
    // Cache miss - get from catalog
    price, err := h.catalog.GetPrice(ctx, query.ProductID)
    if err != nil {
        return err
    }
    
    // Cache the result
    h.cache.Set(ctx, cacheKey, price, h.ttl)
    
    // Return response
    // ...
}
```

</td>
</tr>
</table>

## Monitoring

### Metrics

Key metrics to track:
- Request rate
- Response time (p50, p95, p99)
- Timeout rate
- Error rate
- Queue depth
- Active correlations

### Health Checks

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class BridgeHealthCheck : IHealthCheck
{
    private readonly ISyncAsyncBridge _bridge;
    
    public async Task<HealthCheckResult> 
        CheckHealthAsync(
            HealthCheckContext context,
            CancellationToken ct)
    {
        try
        {
            // Send health check query
            var reply = await _bridge
                .SendAndWaitAsync<HealthReply>(
                    new HealthQuery(),
                    timeout: TimeSpan.FromSeconds(5),
                    cancellationToken: ct);
            
            return reply.IsHealthy
                ? HealthCheckResult.Healthy()
                : HealthCheckResult.Unhealthy(
                    reply.Message);
        }
        catch (TimeoutException)
        {
            return HealthCheckResult.Unhealthy(
                "Health check timed out");
        }
    }
}
```

</td>
<td>

```go
type BridgeHealthCheck struct {
    bridge *bridge.SyncAsyncBridge
}

func (h *BridgeHealthCheck) CheckHealth(
    ctx context.Context) error {
    
    // Send health check query
    ctx, cancel := context.WithTimeout(
        ctx, 5*time.Second)
    defer cancel()
    
    reply, err := h.bridge.SendAndWait(ctx,
        &HealthQuery{
            BaseQuery: contracts.NewBaseQuery(
                "HealthQuery"),
        },
        "qry.health.check",
        5*time.Second)
    
    if err != nil {
        var timeoutErr *bridge.TimeoutError
        if errors.As(err, &timeoutErr) {
            return errors.New("health check timed out")
        }
        return err
    }
    
    healthReply := reply.(*HealthReply)
    if !healthReply.IsHealthy {
        return errors.New(healthReply.Message)
    }
    
    return nil
}
```

</td>
</tr>
</table>

## Best Practices

1. **Timeout Configuration**
   - Set realistic timeouts based on expected response times
   - Consider network latency and processing time
   - Use shorter timeouts for queries than commands
   - Implement timeout handlers for graceful degradation

2. **Error Handling**
   - Always handle timeout scenarios
   - Distinguish between transient and permanent failures
   - Provide meaningful error messages in replies
   - Log errors with correlation IDs

3. **Performance**
   - Use connection pooling
   - Implement caching for read-heavy operations
   - Batch requests when possible
   - Monitor queue depths and response times

4. **Reliability**
   - Implement circuit breakers for downstream protection
   - Use retry policies for transient failures
   - Set up proper monitoring and alerting
   - Test failure scenarios

5. **Security**
   - Validate all incoming requests
   - Use authentication headers
   - Implement authorization in handlers
   - Never expose sensitive data in error messages

## Common Use Cases

### Service-to-Service Communication
Replace HTTP calls with reliable messaging:
- Better fault tolerance
- Automatic retries
- Location transparency
- Load balancing

### API Gateway Pattern
Bridge between synchronous HTTP and async messaging:
- HTTP endpoints call Bridge
- Maintains REST-like interface
- Benefits of message queuing

### Legacy System Integration
Wrap async systems with synchronous interface:
- Easier migration path
- Familiar programming model
- Gradual modernization

## Next Steps

- Implement [Interceptors](../interceptors/README.md) for cross-cutting concerns
- Add [Monitoring](../monitoring/README.md) to track performance
- Review [Examples](../../README.md#examples) for real-world usage
- Learn about [Performance Tuning](../../advanced/performance.md)