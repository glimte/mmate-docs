# Response Tracking - Request-Response Lifecycle

This document covers the complete request-response lifecycle, correlation ID tracking, and response pattern implementation in Mmate.

## Overview

Response tracking enables:
- Correlation of requests with their responses
- Timeout handling for unresponsive services
- Multiple response aggregation
- Response routing to the correct caller
- Distributed tracing support

### Full Request-Response Lifecycle

The complete lifecycle of a request typically involves:

1. App A sends a request to App B with a correlation ID
2. App B acknowledges receipt (via AutoAcknowledgingMessageHandler)
3. App B processes the request 
4. App B sends a response back to App A
5. App A associates the response with the original request using the correlation ID

## Core Concepts

### Correlation IDs
Every request-response pair is linked by a unique correlation ID that travels with the message throughout its lifecycle.

### Reply Queues
Responses are routed back to the caller through dedicated reply queues, either ephemeral or persistent.

### Response Store
An in-memory or distributed store that holds pending responses until they're collected or timeout.

## Implementation

### Basic Request-Response

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Response tracking service
public class ResponseTracker : IResponseTracker
{
    private readonly ConcurrentDictionary<string, PendingResponse> _pending;
    private readonly ILogger<ResponseTracker> _logger;
    private readonly Timer _cleanupTimer;
    
    public ResponseTracker(ILogger<ResponseTracker> logger)
    {
        _logger = logger;
        _pending = new ConcurrentDictionary<string, PendingResponse>();
        _cleanupTimer = new Timer(Cleanup, null, 
            TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1));
    }
    
    public async Task<TResponse> GetResponseAsync<TResponse>(
        string correlationId, 
        TimeSpan timeout,
        CancellationToken cancellationToken = default)
        where TResponse : class
    {
        var pending = new PendingResponse
        {
            CorrelationId = correlationId,
            Timeout = DateTime.UtcNow.Add(timeout),
            CompletionSource = new TaskCompletionSource<object>(),
            ResponseType = typeof(TResponse)
        };
        
        if (!_pending.TryAdd(correlationId, pending))
        {
            throw new InvalidOperationException(
                $"Correlation ID {correlationId} already exists");
        }
        
        using (cancellationToken.Register(() => 
            pending.CompletionSource.TrySetCanceled()))
        {
            try
            {
                var response = await pending.CompletionSource.Task
                    .ConfigureAwait(false);
                    
                if (response is TResponse typed)
                {
                    return typed;
                }
                
                throw new InvalidCastException(
                    $"Expected {typeof(TResponse).Name} but got {response?.GetType().Name}");
            }
            catch (TaskCanceledException) when (DateTime.UtcNow >= pending.Timeout)
            {
                throw new TimeoutException(
                    $"Request timed out after {timeout}");
            }
            finally
            {
                _pending.TryRemove(correlationId, out _);
            }
        }
    }
    
    public void SetResponse(string correlationId, object response)
    {
        if (_pending.TryGetValue(correlationId, out var pending))
        {
            pending.CompletionSource.TrySetResult(response);
        }
        else
        {
            _logger.LogWarning(
                "Received response for unknown correlation ID: {CorrelationId}", 
                correlationId);
        }
    }
    
    private void Cleanup(object state)
    {
        var expired = _pending
            .Where(kvp => DateTime.UtcNow >= kvp.Value.Timeout)
            .ToList();
        
        foreach (var kvp in expired)
        {
            if (_pending.TryRemove(kvp.Key, out var pending))
            {
                pending.CompletionSource.TrySetCanceled();
            }
        }
        
        if (expired.Any())
        {
            _logger.LogInformation(
                "Cleaned up {Count} expired pending responses", 
                expired.Count);
        }
    }
}

// Pending response holder
public class PendingResponse
{
    public string CorrelationId { get; set; }
    public DateTime Timeout { get; set; }
    public TaskCompletionSource<object> CompletionSource { get; set; }
    public Type ResponseType { get; set; }
}

// Using with message dispatcher
public class EnhancedMessageDispatcher : IMessageDispatcher
{
    private readonly IMessagePublisher _publisher;
    private readonly IResponseTracker _tracker;
    private readonly string _replyQueue;
    
    public EnhancedMessageDispatcher(
        IMessagePublisher publisher,
        IResponseTracker tracker,
        string replyQueue)
    {
        _publisher = publisher;
        _tracker = tracker;
        _replyQueue = replyQueue;
    }
    
    public async Task<TResponse> RequestAsync<TRequest, TResponse>(
        TRequest request, 
        TimeSpan? timeout = null,
        CancellationToken cancellationToken = default)
        where TRequest : class
        where TResponse : class
    {
        var correlationId = Guid.NewGuid().ToString();
        var effectiveTimeout = timeout ?? TimeSpan.FromSeconds(30);
        
        // Add correlation ID and reply queue
        if (request is IMessage message)
        {
            message.CorrelationId = correlationId;
            message.Headers["ReplyTo"] = _replyQueue;
        }
        
        // Start tracking before publishing
        var responseTask = _tracker.GetResponseAsync<TResponse>(
            correlationId, effectiveTimeout, cancellationToken);
        
        // Publish request
        await _publisher.PublishAsync(request, cancellationToken);
        
        // Wait for response
        return await responseTask;
    }
}
```

</td>
<td>

```go
// Response tracking service
type ResponseTracker struct {
    pending map[string]*PendingResponse
    mu      sync.RWMutex
    cleanup *time.Ticker
    logger  *slog.Logger
}

type PendingResponse struct {
    CorrelationID string
    Timeout       time.Time
    Response      chan interface{}
    ResponseType  reflect.Type
}

func NewResponseTracker(logger *slog.Logger) *ResponseTracker {
    rt := &ResponseTracker{
        pending: make(map[string]*PendingResponse),
        cleanup: time.NewTicker(time.Minute),
        logger:  logger,
    }
    
    go rt.cleanupLoop()
    return rt
}

func (rt *ResponseTracker) GetResponse(
    ctx context.Context,
    correlationID string,
    timeout time.Duration,
    responseType reflect.Type) (interface{}, error) {
    
    pending := &PendingResponse{
        CorrelationID: correlationID,
        Timeout:       time.Now().Add(timeout),
        Response:      make(chan interface{}, 1),
        ResponseType:  responseType,
    }
    
    rt.mu.Lock()
    if _, exists := rt.pending[correlationID]; exists {
        rt.mu.Unlock()
        return nil, fmt.Errorf("correlation ID %s already exists", correlationID)
    }
    rt.pending[correlationID] = pending
    rt.mu.Unlock()
    
    defer func() {
        rt.mu.Lock()
        delete(rt.pending, correlationID)
        rt.mu.Unlock()
    }()
    
    select {
    case response := <-pending.Response:
        // Validate response type
        if reflect.TypeOf(response) != responseType {
            return nil, fmt.Errorf("expected %v but got %T", 
                responseType, response)
        }
        return response, nil
        
    case <-time.After(timeout):
        return nil, fmt.Errorf("request timed out after %v", timeout)
        
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}

func (rt *ResponseTracker) SetResponse(correlationID string, response interface{}) {
    rt.mu.RLock()
    pending, exists := rt.pending[correlationID]
    rt.mu.RUnlock()
    
    if exists {
        select {
        case pending.Response <- response:
            // Response delivered
        default:
            rt.logger.Warn("Failed to deliver response",
                "correlationId", correlationID,
                "reason", "channel full or closed")
        }
    } else {
        rt.logger.Warn("Received response for unknown correlation ID",
            "correlationId", correlationID)
    }
}

func (rt *ResponseTracker) cleanupLoop() {
    for range rt.cleanup.C {
        rt.mu.Lock()
        now := time.Now()
        expired := make([]string, 0)
        
        for id, pending := range rt.pending {
            if now.After(pending.Timeout) {
                expired = append(expired, id)
                close(pending.Response)
            }
        }
        
        for _, id := range expired {
            delete(rt.pending, id)
        }
        rt.mu.Unlock()
        
        if len(expired) > 0 {
            rt.logger.Info("Cleaned up expired responses",
                "count", len(expired))
        }
    }
}

// Using with message dispatcher
type EnhancedMessageDispatcher struct {
    publisher messaging.Publisher
    tracker   *ResponseTracker
    replyQueue string
}

func NewEnhancedMessageDispatcher(
    publisher messaging.Publisher,
    tracker *ResponseTracker,
    replyQueue string) *EnhancedMessageDispatcher {
    
    return &EnhancedMessageDispatcher{
        publisher:  publisher,
        tracker:    tracker,
        replyQueue: replyQueue,
    }
}

func (d *EnhancedMessageDispatcher) Request(
    ctx context.Context,
    request contracts.Message,
    responseType reflect.Type,
    timeout time.Duration) (interface{}, error) {
    
    correlationID := uuid.New().String()
    
    // Add correlation ID and reply queue
    request.SetHeader("correlationId", correlationID)
    request.SetHeader("replyTo", d.replyQueue)
    
    // Start tracking before publishing
    responseChan := make(chan interface{}, 1)
    errChan := make(chan error, 1)
    
    go func() {
        response, err := d.tracker.GetResponse(
            ctx, correlationID, timeout, responseType)
        if err != nil {
            errChan <- err
        } else {
            responseChan <- response
        }
    }()
    
    // Publish request
    if err := d.publisher.Publish(ctx, request); err != nil {
        return nil, err
    }
    
    // Wait for response
    select {
    case response := <-responseChan:
        return response, nil
    case err := <-errChan:
        return nil, err
    }
}
```

</td>
</tr>
</table>

### Multiple Response Aggregation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Aggregate multiple responses
public class ResponseAggregator<TResponse> where TResponse : class
{
    private readonly List<TResponse> _responses = new();
    private readonly int _expectedCount;
    private readonly TaskCompletionSource<IReadOnlyList<TResponse>> _completion;
    private readonly Timer _timeout;
    private readonly object _lock = new();
    
    public ResponseAggregator(int expectedCount, TimeSpan timeout)
    {
        _expectedCount = expectedCount;
        _completion = new TaskCompletionSource<IReadOnlyList<TResponse>>();
        _timeout = new Timer(_ => TimeoutReached(), null, timeout, Timeout.InfiniteTimeSpan);
    }
    
    public void AddResponse(TResponse response)
    {
        lock (_lock)
        {
            _responses.Add(response);
            
            if (_responses.Count >= _expectedCount)
            {
                _timeout?.Dispose();
                _completion.TrySetResult(_responses.AsReadOnly());
            }
        }
    }
    
    public Task<IReadOnlyList<TResponse>> GetResponsesAsync()
    {
        return _completion.Task;
    }
    
    private void TimeoutReached()
    {
        lock (_lock)
        {
            _completion.TrySetResult(_responses.AsReadOnly());
        }
    }
}

// Usage in scatter-gather pattern
public class ScatterGatherService
{
    private readonly IMessagePublisher _publisher;
    private readonly IResponseTracker _tracker;
    
    public async Task<PriceQuote[]> GetBestPriceAsync(
        PriceRequest request,
        TimeSpan timeout)
    {
        var aggregator = new ResponseAggregator<PriceQuote>(
            expectedCount: 5, // Expect 5 suppliers
            timeout: timeout);
        
        var correlationId = Guid.NewGuid().ToString();
        
        // Scatter - send to all suppliers
        await _publisher.PublishAsync(new GetQuoteCommand
        {
            CorrelationId = correlationId,
            ProductId = request.ProductId,
            Quantity = request.Quantity
        });
        
        // Gather responses
        var quotes = await aggregator.GetResponsesAsync();
        
        // Return sorted by price
        return quotes.OrderBy(q => q.Price).ToArray();
    }
}
```

</td>
<td>

```go
// Aggregate multiple responses
type ResponseAggregator[T any] struct {
    responses     []T
    expectedCount int
    completion    chan []T
    timeout       *time.Timer
    mu            sync.Mutex
}

func NewResponseAggregator[T any](
    expectedCount int, 
    timeout time.Duration) *ResponseAggregator[T] {
    
    agg := &ResponseAggregator[T]{
        responses:     make([]T, 0, expectedCount),
        expectedCount: expectedCount,
        completion:    make(chan []T, 1),
    }
    
    agg.timeout = time.AfterFunc(timeout, agg.timeoutReached)
    return agg
}

func (a *ResponseAggregator[T]) AddResponse(response T) {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    a.responses = append(a.responses, response)
    
    if len(a.responses) >= a.expectedCount {
        a.timeout.Stop()
        select {
        case a.completion <- a.responses:
        default:
        }
    }
}

func (a *ResponseAggregator[T]) GetResponses() []T {
    return <-a.completion
}

func (a *ResponseAggregator[T]) timeoutReached() {
    a.mu.Lock()
    defer a.mu.Unlock()
    
    select {
    case a.completion <- a.responses:
    default:
    }
}

// Usage in scatter-gather pattern
type ScatterGatherService struct {
    publisher messaging.Publisher
    tracker   *ResponseTracker
}

func (s *ScatterGatherService) GetBestPrice(
    ctx context.Context,
    request PriceRequest,
    timeout time.Duration) ([]PriceQuote, error) {
    
    aggregator := NewResponseAggregator[*PriceQuote](
        5, // Expect 5 suppliers
        timeout)
    
    correlationID := uuid.New().String()
    
    // Scatter - send to all suppliers
    cmd := &GetQuoteCommand{
        BaseCommand:   contracts.NewBaseCommand("GetQuoteCommand"),
        CorrelationID: correlationID,
        ProductID:     request.ProductID,
        Quantity:      request.Quantity,
    }
    
    if err := s.publisher.PublishCommand(ctx, cmd); err != nil {
        return nil, err
    }
    
    // Set up response collection
    go func() {
        for {
            response, err := s.tracker.GetResponse(
                ctx, correlationID, timeout, 
                reflect.TypeOf(&PriceQuote{}))
            if err != nil {
                return
            }
            
            if quote, ok := response.(*PriceQuote); ok {
                aggregator.AddResponse(quote)
            }
        }
    }()
    
    // Gather responses
    quotes := aggregator.GetResponses()
    
    // Sort by price
    sort.Slice(quotes, func(i, j int) bool {
        return quotes[i].Price < quotes[j].Price
    })
    
    return quotes, nil
}
```

</td>
</tr>
</table>

## Response Patterns

### Request-Reply Pattern

The simplest pattern where one request expects exactly one response.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Query handler that sends replies
public class ProductQueryHandler : IMessageHandler<GetProductQuery>
{
    private readonly IProductRepository _repository;
    private readonly IMessagePublisher _publisher;
    
    public async Task HandleAsync(
        GetProductQuery query, 
        MessageContext context)
    {
        var product = await _repository.GetByIdAsync(query.ProductId);
        
        var reply = new ProductReply
        {
            CorrelationId = query.CorrelationId,
            Success = product != null,
            Product = product,
            Error = product == null ? "Product not found" : null
        };
        
        // Send reply to the reply queue
        var replyTo = context.Headers["ReplyTo"];
        await _publisher.PublishToQueueAsync(replyTo, reply);
    }
}
```

</td>
<td>

```go
// Query handler that sends replies
type ProductQueryHandler struct {
    repository ProductRepository
    publisher  messaging.Publisher
}

func (h *ProductQueryHandler) Handle(
    ctx context.Context, 
    msg contracts.Message) error {
    
    query := msg.(*GetProductQuery)
    product, err := h.repository.GetByID(ctx, query.ProductID)
    
    reply := &ProductReply{
        BaseReply:     contracts.NewBaseReply(),
        CorrelationID: query.GetCorrelationID(),
        Success:       err == nil,
        Product:       product,
    }
    
    if err != nil {
        reply.Error = err.Error()
    }
    
    // Send reply to the reply queue
    replyTo := query.GetHeader("ReplyTo")
    return h.publisher.PublishToQueue(ctx, replyTo, reply)
}
```

</td>
</tr>
</table>

### Request-Stream Pattern

One request triggers a stream of responses.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Streaming responses
public class OrderStatusStreamHandler : 
    IMessageHandler<SubscribeToOrderUpdates>
{
    private readonly IOrderTracker _tracker;
    private readonly IMessagePublisher _publisher;
    
    public async Task HandleAsync(
        SubscribeToOrderUpdates request, 
        MessageContext context)
    {
        var replyTo = context.Headers["ReplyTo"];
        var correlationId = request.CorrelationId;
        
        // Subscribe to order updates
        await foreach (var update in _tracker.GetUpdatesAsync(
            request.OrderId, context.CancellationToken))
        {
            var response = new OrderStatusUpdate
            {
                CorrelationId = correlationId,
                OrderId = request.OrderId,
                Status = update.Status,
                Timestamp = update.Timestamp,
                IsLast = update.IsFinal
            };
            
            await _publisher.PublishToQueueAsync(replyTo, response);
            
            if (update.IsFinal)
            {
                break;
            }
        }
    }
}
```

</td>
<td>

```go
// Streaming responses
type OrderStatusStreamHandler struct {
    tracker   OrderTracker
    publisher messaging.Publisher
}

func (h *OrderStatusStreamHandler) Handle(
    ctx context.Context, 
    msg contracts.Message) error {
    
    request := msg.(*SubscribeToOrderUpdates)
    replyTo := request.GetHeader("ReplyTo")
    correlationID := request.GetCorrelationID()
    
    // Subscribe to order updates
    updates, err := h.tracker.GetUpdates(ctx, request.OrderID)
    if err != nil {
        return err
    }
    
    for update := range updates {
        response := &OrderStatusUpdate{
            BaseEvent:     contracts.NewBaseEvent("OrderStatusUpdate", request.OrderID),
            CorrelationID: correlationID,
            OrderID:       request.OrderID,
            Status:        update.Status,
            Timestamp:     update.Timestamp,
            IsLast:        update.IsFinal,
        }
        
        if err := h.publisher.PublishToQueue(ctx, replyTo, response); err != nil {
            return err
        }
        
        if update.IsFinal {
            break
        }
    }
    
    return nil
}
```

</td>
</tr>
</table>

## Advanced Response Tracking

### Distributed Response Cache

For high-scale scenarios, use a distributed cache for response tracking.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class DistributedResponseTracker : IResponseTracker
{
    private readonly IDistributedCache _cache;
    private readonly IMessageSerializer _serializer;
    
    public async Task<TResponse> GetResponseAsync<TResponse>(
        string correlationId, 
        TimeSpan timeout,
        CancellationToken cancellationToken)
        where TResponse : class
    {
        var key = $"response:{correlationId}";
        var endTime = DateTime.UtcNow.Add(timeout);
        
        while (DateTime.UtcNow < endTime)
        {
            var data = await _cache.GetAsync(key, cancellationToken);
            if (data != null)
            {
                await _cache.RemoveAsync(key, cancellationToken);
                return _serializer.Deserialize<TResponse>(data);
            }
            
            await Task.Delay(100, cancellationToken);
        }
        
        throw new TimeoutException($"Request timed out after {timeout}");
    }
    
    public async Task SetResponseAsync(
        string correlationId, 
        object response,
        CancellationToken cancellationToken)
    {
        var key = $"response:{correlationId}";
        var data = _serializer.Serialize(response);
        
        await _cache.SetAsync(key, data, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
        }, cancellationToken);
    }
}
```

</td>
<td>

```go
type DistributedResponseTracker struct {
    cache      DistributedCache
    serializer MessageSerializer
}

func (rt *DistributedResponseTracker) GetResponse(
    ctx context.Context,
    correlationID string,
    timeout time.Duration,
    responseType reflect.Type) (interface{}, error) {
    
    key := fmt.Sprintf("response:%s", correlationID)
    endTime := time.Now().Add(timeout)
    
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for time.Now().Before(endTime) {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
            
        case <-ticker.C:
            data, err := rt.cache.Get(ctx, key)
            if err != nil {
                return nil, err
            }
            
            if data != nil {
                // Remove from cache
                rt.cache.Delete(ctx, key)
                
                // Deserialize response
                response, err := rt.serializer.Deserialize(
                    data, responseType)
                if err != nil {
                    return nil, err
                }
                
                return response, nil
            }
        }
    }
    
    return nil, fmt.Errorf("request timed out after %v", timeout)
}

func (rt *DistributedResponseTracker) SetResponse(
    ctx context.Context,
    correlationID string,
    response interface{}) error {
    
    key := fmt.Sprintf("response:%s", correlationID)
    data, err := rt.serializer.Serialize(response)
    if err != nil {
        return err
    }
    
    return rt.cache.SetWithTTL(ctx, key, data, 5*time.Minute)
}
```

</td>
</tr>
</table>

## Monitoring Response Tracking

### Metrics to Track

1. **Response Times**
   - Average response time per message type
   - P50, P95, P99 percentiles
   - Timeout rate

2. **Correlation Success**
   - Successful correlations
   - Orphaned responses
   - Duplicate correlation IDs

3. **Memory Usage**
   - Pending response count
   - Memory consumed by response tracker
   - Cleanup effectiveness

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ResponseTrackingMetrics
{
    private readonly IMetricsCollector _metrics;
    
    public void RecordResponseTime(
        string messageType, 
        TimeSpan duration,
        bool success)
    {
        _metrics.RecordHistogram(
            "response_time_seconds",
            duration.TotalSeconds,
            new KeyValuePair<string, object>("type", messageType),
            new KeyValuePair<string, object>("success", success));
    }
    
    public void RecordTimeout(string messageType)
    {
        _metrics.IncrementCounter(
            "response_timeouts_total",
            new KeyValuePair<string, object>("type", messageType));
    }
    
    public void RecordOrphanedResponse(string correlationId)
    {
        _metrics.IncrementCounter(
            "orphaned_responses_total");
    }
}
```

</td>
<td>

```go
type ResponseTrackingMetrics struct {
    responseTime    *prometheus.HistogramVec
    timeouts        *prometheus.CounterVec
    orphanResponses prometheus.Counter
}

func NewResponseTrackingMetrics() *ResponseTrackingMetrics {
    return &ResponseTrackingMetrics{
        responseTime: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name: "mmate_response_time_seconds",
                Help: "Response time in seconds",
            },
            []string{"type", "success"},
        ),
        timeouts: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "mmate_response_timeouts_total",
                Help: "Total number of response timeouts",
            },
            []string{"type"},
        ),
        orphanResponses: prometheus.NewCounter(
            prometheus.CounterOpts{
                Name: "mmate_orphaned_responses_total",
                Help: "Total number of orphaned responses",
            },
        ),
    }
}

func (m *ResponseTrackingMetrics) RecordResponseTime(
    messageType string,
    duration time.Duration,
    success bool) {
    
    m.responseTime.WithLabelValues(
        messageType,
        fmt.Sprintf("%t", success),
    ).Observe(duration.Seconds())
}
```

</td>
</tr>
</table>

## Best Practices

1. **Use Appropriate Timeouts**
   - Set realistic timeouts based on expected response times
   - Consider network latency and processing time
   - Allow for retries within the timeout period

2. **Handle Timeout Gracefully**
   - Provide meaningful error messages
   - Consider fallback mechanisms
   - Log timeout occurrences for monitoring

3. **Clean Up Resources**
   - Remove completed responses from tracking
   - Implement periodic cleanup for expired responses
   - Monitor memory usage of response tracker

4. **Correlation ID Generation**
   - Use UUIDs for uniqueness
   - Include timestamp in correlation ID for debugging
   - Never reuse correlation IDs

5. **Response Validation**
   - Validate response types match expectations
   - Check response integrity
   - Handle malformed responses gracefully

6. **Scalability Considerations**
   - Use distributed cache for high-scale scenarios
   - Partition response tracking by service
   - Monitor and alert on tracking overhead