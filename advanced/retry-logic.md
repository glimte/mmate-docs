# Message Retry Logic

The retry mechanism provides resilience against transient failures when sending or publishing messages, handling temporary network issues, broker unavailability, or other intermittent problems.

## How It Works

The retry mechanism uses an exponential backoff strategy:

1. When a messaging operation fails, it is retried up to a configurable number of times
2. Each retry attempt uses an increasing delay between attempts
3. The delay increases exponentially with each attempt, up to a maximum delay
4. Detailed logging tracks each retry attempt and its outcome

## Default Configuration

| Setting | Default Value | Description |
|---------|---------------|-------------|
| Maximum Retries | 3 | Maximum number of retry attempts |
| Initial Delay | 100ms | Delay before the first retry attempt |
| Maximum Delay | 5000ms (5s) | Maximum delay between retry attempts |
| Backoff Multiplier | 2.0 | Each retry's delay is multiplied by this factor |

## Basic Configuration

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
        // Retry configuration
        options.MaxRetryAttempts = 3;
        options.InitialRetryDelayMs = 100;
        options.MaxRetryDelayMs = 5000;
        options.RetryMultiplier = 2.0;
    });
```

</td>
<td>

```go
config := messaging.Config{
    RabbitMQ: rabbitmq.Config{
        // Retry configuration
        MaxRetryAttempts:   3,
        InitialRetryDelay:  100 * time.Millisecond,
        MaxRetryDelay:      5 * time.Second,
        RetryMultiplier:    2.0,
    },
}

dispatcher, err := messaging.NewDispatcher(config)
```

</td>
</tr>
</table>

## Retry Pattern

The retry logic follows this process:

1. Attempt the operation
2. If it succeeds, return the result
3. If it fails:
   - Increment the attempt counter
   - If max retries reached or exception is non-retriable, throw/return error
   - Otherwise, wait for the calculated delay
   - Increase the delay for the next attempt (using backoff multiplier)
   - Try again

## Operations with Retry Logic

The following message operations include automatic retry logic:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

- `SendAsync<T>` - Sending messages to a specific queue
- `PublishAsync<T>` - Publishing messages to a topic
- `RequestAsync<TQuery, TResponse>` - Query operations
- `SendWithAckAsync<T>` - Commands with acknowledgment

</td>
<td>

- `Send()` - Sending messages to a specific queue
- `Publish()` - Publishing messages to a topic
- `Request()` - Query operations
- `SendWithAck()` - Commands with acknowledgment

</td>
</tr>
</table>

## Custom Retry Policies

### Exponential Backoff

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors()
    .ConfigureRetryPolicies(options =>
    {
        options.UseExponentialBackoff = true;
        options.GlobalMaxRetries = 5;
        options.InitialDelayMs = 1000;
        options.MaxDelayMs = 30000;
        options.BackoffMultiplier = 2.0;
        options.AddJitter = true; // Prevent thundering herd
    });
```

</td>
<td>

```go
retryPolicy := &reliability.ExponentialBackoff{
    InitialInterval: time.Second,
    MaxInterval:     30 * time.Second,
    Multiplier:      2.0,
    MaxRetries:      5,
    Jitter:          true, // Prevent thundering herd
}

config := messaging.Config{
    RetryPolicy: retryPolicy,
}
```

</td>
</tr>
</table>

### Fixed Delay

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors()
    .ConfigureRetryPolicies(options =>
    {
        options.UseExponentialBackoff = false;
        options.GlobalMaxRetries = 3;
        options.FixedDelayMs = 1000; // 1 second between attempts
    });
```

</td>
<td>

```go
retryPolicy := &reliability.FixedDelay{
    Delay:      time.Second,
    MaxRetries: 3,
}

config := messaging.Config{
    RetryPolicy: retryPolicy,
}
```

</td>
</tr>
</table>

### Linear Backoff

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Custom retry policy
public class LinearBackoffPolicy : IRetryPolicy
{
    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        CancellationToken ct)
    {
        for (int attempt = 1; attempt <= 3; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (ShouldRetry(ex, attempt))
            {
                var delay = TimeSpan.FromSeconds(attempt);
                await Task.Delay(delay, ct);
            }
        }
        
        return await operation(); // Final attempt
    }
}
```

</td>
<td>

```go
type LinearBackoff struct {
    BaseInterval time.Duration
    MaxRetries   int
}

func (l *LinearBackoff) Execute(
    ctx context.Context,
    operation func() error) error {
    
    for attempt := 1; attempt <= l.MaxRetries; attempt++ {
        err := operation()
        if err == nil {
            return nil
        }
        
        if !l.shouldRetry(err, attempt) {
            return err
        }
        
        delay := time.Duration(attempt) * l.BaseInterval
        select {
        case <-time.After(delay):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    
    return operation() // Final attempt
}
```

</td>
</tr>
</table>

## Exception Handling

### Retryable vs Non-Retryable Errors

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors()
    .ConfigureRetryPolicies(options =>
    {
        // Custom retry predicate
        options.ShouldRetry = (exception, attempt) =>
        {
            // Retry on transient errors only
            return exception is TimeoutException ||
                   exception is SocketException ||
                   exception is HttpRequestException;
        };
        
        // Different retry counts by error type
        options.RetryCountOverrides[typeof(TimeoutException)] = 5;
        options.RetryCountOverrides[typeof(SocketException)] = 3;
    });
```

</td>
<td>

```go
// Define retryable error types
func isRetryableError(err error) bool {
    var timeoutErr *TimeoutError
    var networkErr *NetworkError
    var tempErr interface{ Temporary() bool }
    
    return errors.As(err, &timeoutErr) ||
           errors.As(err, &networkErr) ||
           (errors.As(err, &tempErr) && tempErr.Temporary())
}

retryPolicy := &reliability.ExponentialBackoff{
    ShouldRetry: isRetryableError,
    MaxRetries:  3,
}
```

</td>
</tr>
</table>

### Circuit Breaker Integration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors()
    .ConfigureRetryPolicies(options =>
    {
        options.GlobalMaxRetries = 3;
    })
    .ConfigureCircuitBreaker(options =>
    {
        options.FailureThreshold = 5;
        options.OpenDuration = TimeSpan.FromSeconds(30);
    });

// Retries happen first, then circuit breaker tracks failures
```

</td>
<td>

```go
// Combine retry with circuit breaker
retryPolicy := &reliability.ExponentialBackoff{
    MaxRetries: 3,
}

circuitBreaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(30*time.Second),
)

// Chain them together
handler := reliability.Chain(retryPolicy, circuitBreaker)
```

</td>
</tr>
</table>

## Per-Message Type Configuration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors()
    .ConfigureRetryPolicies(options =>
    {
        // Different policies per message type
        options.MessageTypeOverrides[typeof(CriticalCommand)] = 
            new RetryPolicy 
            { 
                MaxRetries = 5,
                InitialDelayMs = 500 
            };
            
        options.MessageTypeOverrides[typeof(BackgroundTask)] = 
            new RetryPolicy 
            { 
                MaxRetries = 1,
                InitialDelayMs = 5000 
            };
    });
```

</td>
<td>

```go
// Per-message type retry policies
retryConfig := map[string]reliability.RetryPolicy{
    "CriticalCommand": &reliability.ExponentialBackoff{
        MaxRetries:      5,
        InitialInterval: 500 * time.Millisecond,
    },
    "BackgroundTask": &reliability.FixedDelay{
        MaxRetries: 1,
        Delay:      5 * time.Second,
    },
}

dispatcher := messaging.NewDispatcher(
    messaging.WithPerTypeRetry(retryConfig))
```

</td>
</tr>
</table>

## Monitoring and Logging

### Retry Metrics

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class RetryMetricsInterceptor : MessageInterceptorBase
{
    private readonly IMetrics _metrics;
    
    public override async Task OnAfterConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        var retryCount = context.GetHeader<int>("RetryAttempt");
        if (retryCount > 0)
        {
            _metrics.Increment("message.retry.count", 
                new[] { 
                    ("message_type", context.Message.GetType().Name),
                    ("attempt", retryCount.ToString())
                });
        }
    }
}
```

</td>
<td>

```go
type RetryMetricsInterceptor struct {
    metrics metrics.Client
}

func (i *RetryMetricsInterceptor) After(
    ctx context.Context,
    msg messaging.Message,
    err error) {
    
    if retryCount := GetRetryCount(ctx); retryCount > 0 {
        i.metrics.Increment("message.retry.count",
            metrics.Tags{
                "message_type": msg.Type(),
                "attempt":      fmt.Sprintf("%d", retryCount),
            })
    }
}
```

</td>
</tr>
</table>

### Retry Logging

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Automatic retry logging
services.Configure<LoggerFilterOptions>(options =>
{
    options.Rules.Add(new LoggerFilterRule(
        "Mmate.Messaging.Retry",
        LogLevel.Information,
        (provider, category, logLevel) => true));
});

// Log output:
// [INFO] Retrying operation attempt 2/3 after 200ms delay
// [WARN] All retry attempts exhausted for SendAsync operation
```

</td>
<td>

```go
// Configure retry logging
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

retryPolicy := &reliability.ExponentialBackoff{
    Logger: logger,
}

// Log output:
// {"level":"INFO","msg":"retrying operation","attempt":2,"delay":"200ms"}
// {"level":"WARN","msg":"all retry attempts exhausted","operation":"send"}
```

</td>
</tr>
</table>

## Testing Retry Logic

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task RetryPolicy_ShouldRetryOnTransientError()
{
    // Arrange
    var retryPolicy = new ExponentialBackoffPolicy
    {
        MaxRetries = 3,
        InitialDelay = TimeSpan.FromMilliseconds(10)
    };
    
    var attempts = 0;
    Func<Task<string>> operation = () =>
    {
        attempts++;
        if (attempts < 3)
            throw new TimeoutException("Transient error");
        return Task.FromResult("Success");
    };
    
    // Act
    var result = await retryPolicy.ExecuteAsync(operation);
    
    // Assert
    Assert.AreEqual("Success", result);
    Assert.AreEqual(3, attempts);
}
```

</td>
<td>

```go
func TestRetryPolicy_ShouldRetryOnTransientError(t *testing.T) {
    // Arrange
    retryPolicy := &reliability.ExponentialBackoff{
        MaxRetries:      3,
        InitialInterval: 10 * time.Millisecond,
    }
    
    attempts := 0
    operation := func() error {
        attempts++
        if attempts < 3 {
            return &TransientError{Err: errors.New("timeout")}
        }
        return nil
    }
    
    // Act
    err := retryPolicy.Execute(context.Background(), operation)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, 3, attempts)
}
```

</td>
</tr>
</table>

## Best Practices

1. **Choose Appropriate Retry Counts**
   - Critical operations: 5-10 retries
   - Normal operations: 3 retries
   - Background tasks: 1-2 retries

2. **Use Jitter**
   - Prevents thundering herd problem
   - Distributes load more evenly

3. **Combine with Circuit Breaker**
   - Retry for transient failures
   - Circuit breaker for persistent failures

4. **Monitor Retry Rates**
   - High retry rates may indicate systemic issues
   - Use metrics to track retry patterns

5. **Handle Non-Retryable Errors**
   - Don't retry validation errors
   - Don't retry authentication failures
   - Fail fast on permanent errors