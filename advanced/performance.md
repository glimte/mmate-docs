# Performance Tuning

This guide covers performance optimization techniques for Mmate applications to achieve high throughput and efficient resource utilization.

## Configuration Options

### Connection Performance

Configure connection settings for optimal performance:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateMessaging()
    .WithRabbitMqTransport(options =>
    {
        // Connection pooling
        options.MaxConnections = 5;
        options.MaxChannelsPerConnection = 10;
        
        // Retry configuration
        options.MaxRetryAttempts = 5;
        options.InitialRetryDelayMs = 1000;
        options.RetryMultiplier = 2.0;
    });
```

</td>
<td>

```go
config := rabbitmq.Config{
    // Connection pooling
    MaxConnections:         5,
    MaxChannelsPerConnection: 10,
    
    // Retry configuration
    MaxRetryAttempts: 5,
    InitialRetryDelay: time.Second,
    RetryMultiplier:   2.0,
}
```

</td>
</tr>
</table>

### Concurrency Settings

Control parallel message processing:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Message processing concurrency
services.AddMmateMessaging(options =>
{
    options.MaxConcurrentMessages = 10;
});

// StageFlow concurrency
services.AddStageFlow(options =>
{
    options.MaxStageConcurrency = 5;
});

// Bridge concurrency
services.AddMessagingBridge(options =>
{
    options.MaxConcurrentRequests = 1000;
});
```

</td>
<td>

```go
// Message processing concurrency
messagingConfig := messaging.Config{
    MaxConcurrentMessages: 10,
}

// StageFlow concurrency
stageflowConfig := stageflow.Config{
    MaxStageConcurrency: 5,
}

// Bridge concurrency
bridgeConfig := bridge.Config{
    MaxConcurrentRequests: 1000,
}
```

</td>
</tr>
</table>

## High-Throughput Configuration

For IoT or high-volume scenarios:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateMessaging()
    .WithRabbitMqTransport(options =>
    {
        // High throughput settings
        options.PrefetchCount = 1000;
        options.BatchSize = 100;
        options.EnableCompression = true;
    });

services.AddStageFlow(options =>
{
    options.MaxStageConcurrency = 50;
    options.EnableMetrics = true;
    options.HighThroughputMode = true;
});
```

</td>
<td>

```go
// High throughput settings
config := messaging.Config{
    RabbitMQ: rabbitmq.Config{
        PrefetchCount:    1000,
        BatchSize:        100,
        EnableCompression: true,
    },
}

stageflowConfig := stageflow.Config{
    MaxStageConcurrency: 50,
    EnableMetrics:       true,
    HighThroughputMode:  true,
}
```

</td>
</tr>
</table>

## Prefetch Configuration

### Fair Work Distribution

For evenly distributed workloads:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Fair distribution
rabbit.PrefetchCount = 1;

// Batch processing
rabbit.PrefetchCount = 10;

// High throughput
rabbit.PrefetchCount = 100;
```

</td>
<td>

```go
// Fair distribution
channel.Qos(1, 0, false)

// Batch processing
channel.Qos(10, 0, false)

// High throughput
channel.Qos(100, 0, false)
```

</td>
</tr>
</table>

## Batch Processing

Process multiple messages together:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class BatchOrderProcessor : 
    IMessageHandler<IBatchMessage<OrderCommand>>
{
    public async Task HandleAsync(
        IBatchMessage<OrderCommand> batch,
        CancellationToken ct)
    {
        // Process all messages in one DB transaction
        using var transaction = await _db.BeginTransactionAsync();
        
        foreach (var order in batch.Messages)
        {
            await ProcessOrder(order);
        }
        
        await transaction.CommitAsync();
    }
}
```

</td>
<td>

```go
type BatchOrderProcessor struct {
    db *sql.DB
}

func (p *BatchOrderProcessor) HandleBatch(
    ctx context.Context,
    batch []OrderCommand) error {
    
    // Process all messages in one DB transaction
    tx, err := p.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    for _, order := range batch {
        if err := p.processOrder(tx, order); err != nil {
            return err
        }
    }
    
    return tx.Commit()
}
```

</td>
</tr>
</table>

## Performance Monitoring

### Built-in Performance Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class PerformanceInterceptor : 
    MessageInterceptorBase
{
    public override Task OnBeforeConsumeAsync(
        MessageContext context, 
        CancellationToken ct)
    {
        var stopwatch = Stopwatch.StartNew();
        context.Properties["Timer"] = stopwatch;
        return Task.CompletedTask;
    }

    public override Task OnAfterConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        var stopwatch = context.Properties["Timer"] 
            as Stopwatch;
        stopwatch.Stop();
        
        if (stopwatch.ElapsedMilliseconds > 1000)
        {
            _logger.LogWarning(
                "Slow processing: {Type} took {Ms}ms",
                context.Message.GetType().Name,
                stopwatch.ElapsedMilliseconds);
        }
        
        return Task.CompletedTask;
    }
}
```

</td>
<td>

```go
type PerformanceInterceptor struct {
    logger *slog.Logger
}

func (i *PerformanceInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) context.Context {
    
    return context.WithValue(ctx, "startTime", 
        time.Now())
}

func (i *PerformanceInterceptor) After(
    ctx context.Context,
    msg messaging.Message,
    err error) {
    
    start := ctx.Value("startTime").(time.Time)
    elapsed := time.Since(start)
    
    if elapsed > time.Second {
        i.logger.Warn("Slow processing",
            "type", msg.Type(),
            "duration", elapsed)
    }
}
```

</td>
</tr>
</table>

### Performance Metrics Configuration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.ConfigurePerformanceTracking(options =>
{
    options.SlowProcessingThreshold = 
        TimeSpan.FromSeconds(3);
    options.TrackMemoryUsage = true;
    options.EnableDetailedMetrics = true;
});
```

</td>
<td>

```go
perfConfig := performance.Config{
    SlowProcessingThreshold: 3 * time.Second,
    TrackMemoryUsage:        true,
    EnableDetailedMetrics:   true,
}
```

</td>
</tr>
</table>

## Resource Optimization

### Memory Management

1. **Message Size**: Keep messages small, store large data externally
2. **Prefetch Buffer**: Balance between throughput and memory usage
3. **Connection Pooling**: Reuse connections to reduce overhead

### CPU Optimization

1. **Parallel Processing**: Use appropriate concurrency levels
2. **Batch Operations**: Reduce per-message overhead
3. **Async I/O**: Leverage async operations for I/O-bound work

### Network Optimization

1. **Message Compression**: Enable for large messages
2. **Connection Reuse**: Use connection pooling
3. **Local Clustering**: Deploy consumers close to RabbitMQ

## Throttling and Rate Limiting

Protect downstream systems:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Throttle(
    MaxConcurrent = 5, 
    DelayBetweenCallsMs = 100)]
public class ProcessPaymentCommand : BaseCommand
{
    // Rate-limited command
}

// Or configure globally
services.AddThrottling(options =>
{
    options.DefaultMaxConcurrent = 10;
    options.DefaultDelayMs = 50;
});
```

</td>
<td>

```go
// Rate limiter
limiter := rate.NewLimiter(
    rate.Every(100*time.Millisecond), 5)

func (h *Handler) Handle(
    ctx context.Context, 
    msg Message) error {
    
    // Wait for rate limit
    if err := limiter.Wait(ctx); err != nil {
        return err
    }
    
    // Process message
    return h.process(msg)
}
```

</td>
</tr>
</table>

## Queue Depth Monitoring

Configure thresholds based on throughput:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// High-throughput systems
services.AddHealthMonitoring(options =>
{
    options.QueueWarningThreshold = 10000;
    options.QueueCriticalThreshold = 50000;
});

// Low-throughput systems
services.AddHealthMonitoring(options =>
{
    options.QueueWarningThreshold = 100;
    options.QueueCriticalThreshold = 500;
});
```

</td>
<td>

```go
// High-throughput systems
monitor := health.NewMonitor(
    health.WithQueueWarningThreshold(10000),
    health.WithQueueCriticalThreshold(50000),
)

// Low-throughput systems
monitor := health.NewMonitor(
    health.WithQueueWarningThreshold(100),
    health.WithQueueCriticalThreshold(500),
)
```

</td>
</tr>
</table>

## Benchmarking

### Load Testing

```bash
# Using artillery.io
artillery run load-test.yml

# Load test configuration
config:
  target: "amqp://localhost:5672"
  phases:
    - duration: 60
      arrivalRate: 100
    - duration: 120
      arrivalRate: 1000
```

### Measuring Throughput

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ThroughputMeter
{
    private long _messageCount;
    private DateTime _startTime;
    
    public void RecordMessage()
    {
        Interlocked.Increment(ref _messageCount);
    }
    
    public double GetMessagesPerSecond()
    {
        var elapsed = DateTime.UtcNow - _startTime;
        return _messageCount / elapsed.TotalSeconds;
    }
}
```

</td>
<td>

```go
type ThroughputMeter struct {
    messageCount int64
    startTime    time.Time
}

func (m *ThroughputMeter) RecordMessage() {
    atomic.AddInt64(&m.messageCount, 1)
}

func (m *ThroughputMeter) GetMessagesPerSecond() float64 {
    elapsed := time.Since(m.startTime)
    count := atomic.LoadInt64(&m.messageCount)
    return float64(count) / elapsed.Seconds()
}
```

</td>
</tr>
</table>

## Performance Tuning Checklist

1. **Connection Settings**
   - [ ] Configure connection pooling
   - [ ] Set appropriate retry policies
   - [ ] Enable circuit breaker

2. **Concurrency**
   - [ ] Set MaxConcurrentMessages based on workload
   - [ ] Configure StageFlow concurrency
   - [ ] Tune bridge concurrent requests

3. **Message Processing**
   - [ ] Optimize prefetch count
   - [ ] Enable batching where appropriate
   - [ ] Implement throttling for rate-sensitive operations

4. **Monitoring**
   - [ ] Enable performance interceptors
   - [ ] Set up queue depth monitoring
   - [ ] Configure slow processing alerts

5. **Resource Usage**
   - [ ] Monitor memory consumption
   - [ ] Track CPU utilization
   - [ ] Measure network bandwidth

## Best Practices

1. **Start Conservative**: Begin with lower concurrency and increase gradually
2. **Monitor Everything**: You can't optimize what you don't measure
3. **Test Under Load**: Performance characteristics change under load
4. **Consider Downstream**: Don't overwhelm dependent services
5. **Profile First**: Identify bottlenecks before optimizing