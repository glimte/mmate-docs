# Message Retry Logic

> **⚠️ Platform Differences**: Advanced retry features are primarily available in the Go implementation. The .NET implementation provides basic retry capabilities only.

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
<th>.NET (Basic Retry Only)</th>
<th>Go (Full Enterprise Retry)</th>
</tr>
<tr>
<td>

```csharp
// .NET implementation has basic retry in SimpleRetryScheduler
services.AddMmateMessaging(options =>
{
    options.RabbitMqConnection = "amqp://localhost";
});

services.ConfigureRetryPolicies(options =>
{
    // Basic retry options available
    options.MaxRetries = 3;
    options.InitialDelay = TimeSpan.FromMilliseconds(100);
});
```

> **Note**: Advanced retry configurations like TTL-based retries, per-message-type policies, and circuit breaker integration are not available in .NET.

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

<table>
<tr>
<th>.NET (Limited)</th>
<th>Go (Full Support)</th>
</tr>
<tr>
<td>

- `PublishAsync()` - Basic publishing with simple retry
- `PublishToTopicAsync()` - Basic topic publishing
- Basic request/reply via SyncAsyncBridge

**Not Available:**
- ❌ `SendWithAckAsync()` - Not implemented
- ❌ Advanced acknowledgment tracking
- ❌ Per-message-type retry policies

</td>
<td>

- `Send()` - Sending messages to a specific queue
- `Publish()` - Publishing messages to a topic
- `Request()` - Query operations
- `SendWithAck()` - Commands with acknowledgment
- Full retry policy configuration
- TTL-based persistent retries

</td>
</tr>
</table>

## Custom Retry Policies

> **⚠️ .NET Limitation**: Custom retry policies, exponential backoff configuration, and advanced retry features are only available in the Go implementation.

### Go Implementation - Full Retry Features

```go
// Exponential backoff with jitter
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

### Fixed Delay (Go Only)

```go
retryPolicy := &reliability.FixedDelay{
    Delay:      time.Second,
    MaxRetries: 3,
}

config := messaging.Config{
    RetryPolicy: retryPolicy,
}
```

### Linear Backoff (Go Only)

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

## .NET Implementation Limitations

The .NET implementation provides only basic retry functionality through `SimpleRetryScheduler`:

- ✅ Basic in-memory retry for transient failures
- ✅ Simple exponential backoff (fixed configuration)
- ❌ No custom retry policies
- ❌ No per-message-type configuration  
- ❌ No TTL-based persistent retries
- ❌ No circuit breaker integration
- ❌ No advanced monitoring or metrics

**For enterprise retry features, use the Go implementation.**

## Exception Handling (Go Implementation)

### Retryable vs Non-Retryable Errors

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

### Advanced Features (Go Only)

The Go implementation provides enterprise-grade retry features:

- **Circuit Breaker Integration**: Combine retry with circuit breakers
- **Per-Message-Type Policies**: Different retry settings per message type  
- **Advanced Monitoring**: Comprehensive retry metrics and logging
- **TTL-Based Retries**: Persistent retry scheduling using RabbitMQ TTL
- **Jitter Support**: Prevent thundering herd problems
- **Custom Error Classification**: Define which errors should be retried

```go
// Example: Combine retry with circuit breaker
retryPolicy := &reliability.ExponentialBackoff{MaxRetries: 3}
circuitBreaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(30*time.Second),
)
handler := reliability.Chain(retryPolicy, circuitBreaker)
```

For detailed Go examples and implementation, see the [TTL Retry Scheduler](ttl-retry-scheduler.md) documentation.

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