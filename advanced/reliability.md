# Reliability Patterns

Building reliable distributed systems requires careful consideration of failure modes and recovery strategies. This guide covers key reliability patterns in Mmate.

## Automatic Queue Recovery

Mmate provides built-in automatic recovery mechanisms to ensure message processing continues even after connection failures, service restarts, or pod crashes.

### Connection Recovery

Both platforms support automatic connection recovery with exponential backoff:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure automatic recovery
services.AddMmateMessaging()
    .WithRabbitMqTransport(options =>
    {
        // Connection resilience
        options.EnableAutomaticRecovery = true;
        options.MaxRetryAttempts = 5;
        options.InitialRetryDelayMs = 1000;
        options.MaxRetryDelayMs = 30000;
        options.RetryMultiplier = 2.0;
        
        // Circuit breaker integration
        options.EnableCircuitBreaker = true;
        options.CircuitBreakerFailureThreshold = 5;
        options.CircuitBreakerOpenTimeoutSeconds = 30;
    });
```

</td>
<td>

```go
// Automatic recovery is built-in
conn := rabbitmq.NewConnection(
    rabbitmq.WithReconnectDelay(time.Second),
    rabbitmq.WithMaxRetries(5), // 0 = infinite
    rabbitmq.WithConnectionStateListener(
        func(oldState, newState ConnectionState) {
            log.Printf("Connection: %s -> %s", 
                oldState, newState)
        }),
)

// Circuit breaker configuration
cb := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(30*time.Second),
)
```

</td>
</tr>
</table>

### Message State Persistence

Mmate's StageFlow provides queue-based state persistence, allowing workflows to survive service crashes without losing progress:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// State travels with the message
public class FlowMessageEnvelope<T>
{
    // Complete workflow state
    public string SerializedWorkflowState { get; set; }
    public string WorkflowStateType { get; set; }
    
    // Processing checkpoints
    public ProcessingCheckpoint Checkpoint { get; set; }
}

// Usage in workflow
workflow.Stage<OrderCommand>(async (context, state, cmd) => 
{
    // Check if resuming from crash
    if (context.IsStepCompleted("validation"))
    {
        logger.LogInfo("Resuming from checkpoint");
        return;
    }
    
    // Validate order
    await ValidateOrder(cmd);
    
    // Save checkpoint
    await context.CheckpointStepAsync("validation");
});
```

</td>
<td>

```go
// State persistence in workflow
type WorkflowState struct {
    Status       string                 `json:"status"`
    StageResults map[string]interface{} `json:"stageResults"`
    GlobalData   map[string]interface{} `json:"globalData"`
    CurrentStage string                 `json:"currentStage"`
    Version      int                    `json:"version"`
}

// Automatic state saving after each stage
flow := stageflow.NewFlow[*OrderContext](
    "order-processing",
    stageflow.WithStateStore(store))

// State is automatically persisted between stages
flow.AddStage("process", func(ctx *StageContext) error {
    // State loaded from previous stage
    if ctx.State.StageResults["validation"] != nil {
        log.Info("Previous validation found")
    }
    
    // Process...
    
    // State automatically saved for next stage
    return nil
})
```

</td>
</tr>
</table>

### Processing Checkpoints

Fine-grained checkpoints allow resuming within a stage after crashes:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ProcessingCheckpoint
{
    // Completed steps within current stage
    public HashSet<string> CompletedSteps { get; set; }
    
    // Intermediate results from steps
    public Dictionary<string, object> StepResults { get; set; }
    
    // Number of attempts at this stage
    public int StageAttempt { get; set; }
    
    // Timestamp of last update
    public DateTimeOffset LastUpdated { get; set; }
}

// Usage
await context.CheckpointStepAsync("inventory-check");
await context.StoreStepResultAsync("inventory", result);

// After crash, resume from checkpoint
if (!context.IsStepCompleted("payment"))
{
    var invResult = context.GetStepResult<InventoryResult>("inventory");
    // Continue processing...
}
```

</td>
<td>

```go
// DLQ handler provides recovery mechanisms
type DLQHandler struct {
    maxRetries int
    retryDelay time.Duration
    errorStore ErrorStore
}

// Automatic retry of failed messages
func (h *DLQHandler) ProcessDLQMessage(msg Message) error {
    metadata := msg.GetDLQMetadata()
    
    if metadata.RetryCount < h.maxRetries {
        // Retry the message
        return h.retryMessage(msg)
    }
    
    // Store permanently failed message
    return h.errorStore.Store(FailedMessage{
        ID:           msg.ID,
        OriginalQueue: metadata.OriginalQueue,
        Error:        metadata.LastError,
        Content:      msg.Body,
        Timestamp:    time.Now(),
    })
}
```

</td>
</tr>
</table>

### Benefits

1. **Zero Downtime Deployments**: Messages continue processing after service restarts
2. **Crash Recovery**: Workflows resume from last checkpoint after pod crashes
3. **No Lost Messages**: State persistence ensures messages aren't lost during failures
4. **Automatic Reconnection**: Connection failures are handled transparently
5. **Progressive Retry**: Exponential backoff prevents overwhelming recovering services

## Circuit Breaker Pattern

### Overview

Circuit breakers prevent cascading failures by failing fast when a service is unavailable.

### States

1. **Closed** - Normal operation, requests pass through
2. **Open** - Failure threshold exceeded, requests fail immediately
3. **Half-Open** - Testing if service has recovered

### Implementation

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
    options.CircuitBreaker = new CircuitBreakerOptions
    {
        FailureThreshold = 5,
        SamplingDuration = TimeSpan.FromMinutes(1),
        MinimumThroughput = 10,
        DurationOfBreak = TimeSpan.FromSeconds(30),
        OnOpen = (name, duration) => 
            logger.LogWarning($"Circuit opened for {name}"),
        OnReset = name => 
            logger.LogInformation($"Circuit reset for {name}")
    };
});
```

</td>
<td>

```go
breaker := circuit.NewBreaker(
    circuit.WithFailureThreshold(5),
    circuit.WithSamplingDuration(time.Minute),
    circuit.WithMinimumThroughput(10),
    circuit.WithTimeout(30*time.Second),
    circuit.WithOnStateChange(func(from, to circuit.State) {
        log.Printf("Circuit: %s -> %s", from, to)
    }),
)
```

</td>
</tr>
</table>

## Retry Strategies

### Retry Policies

1. **Fixed Delay** - Same delay between attempts
2. **Exponential Backoff** - Increasing delays
3. **Exponential Backoff with Jitter** - Prevents thundering herd

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Exponential backoff
services.AddRetryPolicy(policy => policy
    .Handle<TransientException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => 
            TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (outcome, timespan, count, context) =>
            logger.LogWarning(
                $"Retry {count} after {timespan}ms")));

// With jitter
var jitterer = new Random();
policy.WaitAndRetryAsync(
    retryCount: 3,
    sleepDurationProvider: attempt => 
        TimeSpan.FromSeconds(Math.Pow(2, attempt)) + 
        TimeSpan.FromMilliseconds(jitterer.Next(0, 100)));
```

</td>
<td>

```go
// Exponential backoff
retry := &ExponentialBackoff{
    InitialInterval: time.Second,
    MaxInterval:     30 * time.Second,
    Multiplier:      2,
    MaxRetries:      3,
}

// With jitter
func (r *ExponentialBackoff) NextInterval(n int) time.Duration {
    interval := r.InitialInterval * 
        time.Duration(math.Pow(r.Multiplier, float64(n)))
    
    if interval > r.MaxInterval {
        interval = r.MaxInterval
    }
    
    // Add jitter (±10%)
    jitter := time.Duration(rand.Int63n(
        int64(interval) / 5))
    return interval + jitter - interval/10
}
```

</td>
</tr>
</table>

## Dead Letter Queue (DLQ) Handling

### When Messages Go to DLQ

1. **Processing failures** - Unhandled exceptions
2. **Message rejection** - Invalid format/content
3. **TTL expiration** - Message too old
4. **Retry exhaustion** - Max retries exceeded

### DLQ Strategies

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure DLQ
services.AddMmate(options =>
{
    options.DeadLetterExchange = "mmate.dlx";
    options.DeadLetterStrategy = new DeadLetterStrategy
    {
        MaxRetries = 3,
        RetryDelay = TimeSpan.FromMinutes(5),
        OnDeadLetter = async (message, reason) =>
        {
            await alertService.NotifyDeadLetter(
                message, reason);
            await deadLetterStore.Store(message);
        }
    };
});

// Requeue from DLQ
public async Task<int> RequeueDeadLetters(
    string queueName, 
    int batchSize = 100)
{
    var dlqName = $"dlq.{queueName}";
    var messages = await GetMessages(
        dlqName, batchSize);
    
    var requeuedCount = 0;
    foreach (var message in messages)
    {
        try
        {
            // Remove DLQ headers
            message.Headers.Remove("x-death");
            
            // Republish to original queue
            await publisher.PublishAsync(
                message, "", queueName);
            
            // Acknowledge from DLQ
            await channel.BasicAckAsync(
                message.DeliveryTag, false);
            
            requeuedCount++;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, 
                "Failed to requeue message {MessageId}", 
                message.MessageId);
        }
    }
    
    return requeuedCount;
}
```

</td>
<td>

```go
// Configure DLQ
type DLQConfig struct {
    Exchange    string
    MaxRetries  int
    RetryDelay  time.Duration
    OnDeadLetter func(msg Message, reason string) error
}

config := DLQConfig{
    Exchange:   "mmate.dlx",
    MaxRetries: 3,
    RetryDelay: 5 * time.Minute,
    OnDeadLetter: func(msg Message, reason string) error {
        if err := alertService.NotifyDeadLetter(
            msg, reason); err != nil {
            return err
        }
        return deadLetterStore.Store(msg)
    },
}

// Requeue from DLQ
func (d *DLQHandler) RequeueDeadLetters(
    ctx context.Context,
    queueName string,
    batchSize int) (int, error) {
    
    dlqName := fmt.Sprintf("dlq.%s", queueName)
    messages, err := d.getMessages(
        ctx, dlqName, batchSize)
    if err != nil {
        return 0, err
    }
    
    requeuedCount := 0
    for _, msg := range messages {
        // Remove DLQ headers
        delete(msg.Headers, "x-death")
        
        // Republish to original queue
        err := d.publisher.Publish(ctx, msg,
            WithExchange(""),
            WithRoutingKey(queueName))
        
        if err != nil {
            log.Printf("Failed to requeue %s: %v",
                msg.ID, err)
            continue
        }
        
        // Acknowledge from DLQ
        if err := msg.Ack(); err != nil {
            return requeuedCount, err
        }
        
        requeuedCount++
    }
    
    return requeuedCount, nil
}
```

</td>
</tr>
</table>

## Timeout Patterns

### Types of Timeouts

1. **Connection timeout** - Initial connection establishment
2. **Request timeout** - Individual request processing
3. **Circuit breaker timeout** - How long to stay open
4. **Retry timeout** - Maximum time for all retries

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Layered timeouts
services.AddMmate(options =>
{
    // Connection timeout
    options.ConnectionTimeout = 
        TimeSpan.FromSeconds(30);
    
    // Default request timeout
    options.DefaultRequestTimeout = 
        TimeSpan.FromSeconds(60);
    
    // Handler timeout wrapper
    options.AddInterceptor<TimeoutInterceptor>(
        config => config.Timeout = 
            TimeSpan.FromSeconds(45));
});

// Per-request timeout
var cts = new CancellationTokenSource(
    TimeSpan.FromSeconds(30));

try
{
    var result = await bridge
        .SendAndWaitAsync<Reply>(
            command, cts.Token);
}
catch (OperationCanceledException)
{
    logger.LogWarning("Request timed out");
}
```

</td>
<td>

```go
// Layered timeouts
config := Config{
    // Connection timeout
    ConnectionTimeout: 30 * time.Second,
    
    // Default request timeout
    DefaultRequestTimeout: 60 * time.Second,
}

// Context with timeout
ctx, cancel := context.WithTimeout(
    context.Background(), 30*time.Second)
defer cancel()

// Handler timeout wrapper
type TimeoutHandler struct {
    handler Handler
    timeout time.Duration
}

func (h *TimeoutHandler) Handle(
    ctx context.Context, 
    msg Message) error {
    
    ctx, cancel := context.WithTimeout(
        ctx, h.timeout)
    defer cancel()
    
    done := make(chan error, 1)
    go func() {
        done <- h.handler.Handle(ctx, msg)
    }()
    
    select {
    case err := <-done:
        return err
    case <-ctx.Done():
        return fmt.Errorf("handler timeout: %w", 
            ctx.Err())
    }
}
```

</td>
</tr>
</table>

## Idempotency

### Ensuring Exactly-Once Processing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class IdempotentHandler<T> : 
    IMessageHandler<T> where T : IMessage
{
    private readonly IIdempotencyStore _store;
    private readonly IMessageHandler<T> _inner;
    
    public async Task HandleAsync(
        T message, 
        MessageContext context)
    {
        var key = GetIdempotencyKey(message);
        
        // Check if already processed
        var result = await _store
            .GetResultAsync(key);
        if (result != null)
        {
            logger.LogInformation(
                "Skipping duplicate message {Key}", 
                key);
            return;
        }
        
        // Process and store result
        try
        {
            await _inner.HandleAsync(
                message, context);
            
            await _store.StoreResultAsync(
                key, "success", 
                TimeSpan.FromHours(24));
        }
        catch (Exception ex)
        {
            await _store.StoreResultAsync(
                key, "failed:" + ex.Message,
                TimeSpan.FromHours(1));
            throw;
        }
    }
}
```

</td>
<td>

```go
type IdempotentHandler struct {
    store  IdempotencyStore
    inner  Handler
    logger *slog.Logger
}

func (h *IdempotentHandler) Handle(
    ctx context.Context, 
    msg Message) error {
    
    key := h.getIdempotencyKey(msg)
    
    // Check if already processed
    result, err := h.store.GetResult(ctx, key)
    if err != nil {
        return err
    }
    
    if result != nil {
        h.logger.Info("Skipping duplicate",
            "key", key)
        return nil
    }
    
    // Process and store result
    err = h.inner.Handle(ctx, msg)
    
    if err != nil {
        // Store failure (shorter TTL)
        h.store.StoreResult(ctx, key,
            fmt.Sprintf("failed:%v", err),
            time.Hour)
        return err
    }
    
    // Store success
    return h.store.StoreResult(ctx, key,
        "success", 24*time.Hour)
}
```

</td>
</tr>
</table>

## Health Monitoring

### Liveness vs Readiness

- **Liveness**: Is the process alive?
- **Readiness**: Can it handle requests?

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Liveness - basic check
app.MapHealthChecks("/health/live", 
    new HealthCheckOptions
    {
        Predicate = _ => false // No checks
    });

// Readiness - full checks
app.MapHealthChecks("/health/ready", 
    new HealthCheckOptions
    {
        Predicate = check => 
            check.Tags.Contains("ready")
    });

services.AddHealthChecks()
    .AddRabbitMQ(tags: new[] { "ready" })
    .AddCheck<DatabaseHealthCheck>(
        "database", 
        tags: new[] { "ready" });
```

</td>
<td>

```go
// Liveness - basic check
http.HandleFunc("/health/live", 
    func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })

// Readiness - full checks
http.HandleFunc("/health/ready",
    func(w http.ResponseWriter, r *http.Request) {
        checks := []HealthCheck{
            rabbitMQCheck,
            databaseCheck,
        }
        
        for _, check := range checks {
            if err := check.Check(r.Context()); 
               err != nil {
                w.WriteHeader(
                    http.StatusServiceUnavailable)
                json.NewEncoder(w).Encode(
                    map[string]string{
                        "status": "not ready",
                        "error":  err.Error(),
                    })
                return
            }
        }
        
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(
            map[string]string{"status": "ready"})
    })
```

</td>
</tr>
</table>

## Graceful Shutdown

### Handling Shutdown Signals

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        await subscriber.SubscribeAsync(
            "orders",
            HandleMessage,
            stoppingToken);
    }
    
    public override async Task StopAsync(
        CancellationToken cancellationToken)
    {
        logger.LogInformation(
            "Graceful shutdown initiated");
        
        // Stop accepting new messages
        await subscriber.StopAsync();
        
        // Wait for in-flight messages
        var timeout = TimeSpan.FromSeconds(30);
        var gracefulStop = Task.Run(async () =>
        {
            while (subscriber.InFlightCount > 0 && 
                   !cancellationToken
                       .IsCancellationRequested)
            {
                await Task.Delay(100);
            }
        });
        
        await Task.WhenAny(
            gracefulStop,
            Task.Delay(timeout));
        
        await base.StopAsync(cancellationToken);
    }
}
```

</td>
<td>

```go
func main() {
    ctx, cancel := context.WithCancel(
        context.Background())
    defer cancel()
    
    // Handle shutdown signals
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, 
        os.Interrupt, 
        syscall.SIGTERM)
    
    // Start subscriber
    subscriber := messaging.NewSubscriber()
    go subscriber.Start(ctx)
    
    // Wait for shutdown
    <-sigChan
    log.Println("Graceful shutdown initiated")
    
    // Create shutdown context with timeout
    shutdownCtx, shutdownCancel := 
        context.WithTimeout(
            context.Background(), 
            30*time.Second)
    defer shutdownCancel()
    
    // Stop accepting new messages
    cancel()
    
    // Wait for in-flight messages
    done := make(chan struct{})
    go func() {
        for subscriber.InFlightCount() > 0 {
            time.Sleep(100 * time.Millisecond)
        }
        close(done)
    }()
    
    select {
    case <-done:
        log.Println("Graceful shutdown complete")
    case <-shutdownCtx.Done():
        log.Println("Shutdown timeout exceeded")
    }
}
```

</td>
</tr>
</table>

## Best Practices

1. **Fail Fast**
   - Use circuit breakers to prevent cascading failures
   - Set reasonable timeouts
   - Don't retry non-transient errors

2. **Monitor Everything**
   - Track retry rates
   - Monitor DLQ growth
   - Alert on circuit breaker state changes

3. **Design for Idempotency**
   - Use unique message IDs
   - Store processing results
   - Handle duplicates gracefully

4. **Plan for Failure**
   - Test failure scenarios
   - Document recovery procedures
   - Automate where possible

5. **Graceful Degradation**
   - Continue operating with reduced functionality
   - Prioritize critical operations
   - Communicate status clearly

## Testing Reliability

### Chaos Engineering

- Randomly kill processes
- Introduce network delays
- Simulate service failures
- Test recovery procedures

### Load Testing with Failures

```bash
# Simulate failures during load test
while true; do
    # Run load test
    artillery run load-test.yml &
    LOAD_PID=$!
    
    # Wait random time
    sleep $((RANDOM % 60 + 30))
    
    # Inject failure
    docker stop rabbitmq
    sleep 10
    docker start rabbitmq
    
    # Wait for load test to complete
    wait $LOAD_PID
done
```

## Monitoring Reliability Metrics

Key metrics to track:
- Circuit breaker state changes
- Retry attempts and success rates
- DLQ message counts and age
- Message processing latency percentiles
- Error rates by type
- Recovery time after failures