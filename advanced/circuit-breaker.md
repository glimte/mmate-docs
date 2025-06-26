# Circuit Breaker Pattern

The Circuit Breaker pattern is a critical resilience mechanism that prevents cascading failures in distributed systems. When a service or connection experiences repeated failures, the circuit breaker "opens" to block further attempts, giving the failing component time to recover.

## How It Works

The circuit breaker operates in three states:

### 1. Closed State (Normal Operation)
- All operations are allowed to proceed
- Failures are tracked and counted
- If failure count exceeds threshold, transitions to Open state

### 2. Open State (Blocking Operations)
- All operations are immediately rejected with `CircuitBreakerOpenException`
- No actual connection attempts are made
- After a configured timeout, transitions to Half-Open state

### 3. Half-Open State (Testing Recovery)
- Allows a single operation to test if the service has recovered
- If successful, transitions back to Closed state
- If it fails, transitions back to Open state

## Configuration

### Basic Configuration

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
        options.ConnectionString = "amqp://guest:guest@localhost:5672/";
        
        // Enable circuit breaker
        options.EnableCircuitBreaker = true;
        options.CircuitBreakerFailureThreshold = 5;
        options.CircuitBreakerOpenTimeoutSeconds = 30;
    });
```

</td>
<td>

```go
config := messaging.Config{
    RabbitMQ: rabbitmq.Config{
        URL: "amqp://guest:guest@localhost:5672/",
        
        // Circuit breaker configuration
        CircuitBreaker: &reliability.CircuitBreakerConfig{
            Enabled:           true,
            FailureThreshold:  5,
            OpenTimeout:       30 * time.Second,
        },
    },
}

dispatcher, err := messaging.NewDispatcher(config)
```

</td>
</tr>
</table>

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| Enabled/EnableCircuitBreaker | `true` | Enables or disables the circuit breaker |
| FailureThreshold | `5` | Number of failures before circuit opens |
| OpenTimeout | `30s` | Time to wait before trying half-open state |

## Integration with Connection Management

The circuit breaker is integrated with connection management:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class RabbitMqConnectionManager
{
    // Circuit breaker protects connection establishment
    private async Task<IConnection> EstablishConnectionAsync(
        CancellationToken cancellationToken)
    {
        if (_circuitBreaker != null)
        {
            return await _circuitBreaker.ExecuteAsync(async () =>
            {
                return await _factory.CreateConnectionAsync(
                    cancellationToken);
            });
        }
        
        // Fallback to direct connection
        return await _factory.CreateConnectionAsync(
            cancellationToken);
    }
}
```

</td>
<td>

```go
type ConnectionManager struct {
    circuitBreaker *reliability.CircuitBreaker
    factory        ConnectionFactory
}

func (cm *ConnectionManager) EstablishConnection(
    ctx context.Context) (*amqp.Connection, error) {
    
    if cm.circuitBreaker != nil {
        return cm.circuitBreaker.Execute(ctx, func() (*amqp.Connection, error) {
            return cm.factory.CreateConnection(ctx)
        })
    }
    
    // Fallback to direct connection
    return cm.factory.CreateConnection(ctx)
}
```

</td>
</tr>
</table>

## Monitoring Circuit Breaker State

### Accessing Current State

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
var connectionManager = serviceProvider
    .GetService<RabbitMqConnectionManager>();
var state = connectionManager.CircuitBreakerState;

switch (state)
{
    case CircuitBreakerState.Closed:
        logger.LogInformation("Circuit breaker is closed");
        break;
    case CircuitBreakerState.Open:
        logger.LogWarning("Circuit breaker is open");
        break;
    case CircuitBreakerState.HalfOpen:
        logger.LogInformation("Circuit breaker is half-open");
        break;
}
```

</td>
<td>

```go
state := circuitBreaker.State()

switch state {
case reliability.StateClosed:
    log.Info("Circuit breaker is closed")
case reliability.StateOpen:
    log.Warn("Circuit breaker is open")
case reliability.StateHalfOpen:
    log.Info("Circuit breaker is half-open")
}
```

</td>
</tr>
</table>

### Subscribing to State Changes

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Circuit breaker events are logged automatically
services.Configure<LoggerFilterOptions>(options =>
{
    options.Rules.Add(new LoggerFilterRule(
        "Mmate.Messaging.Implementation.CircuitBreaker",
        LogLevel.Information,
        (provider, category, logLevel) => true));
});
```

</td>
<td>

```go
// Subscribe to state changes
circuitBreaker.OnStateChange(func(from, to reliability.State) {
    log.Info("Circuit breaker state changed", 
        "from", from, "to", to)
})
```

</td>
</tr>
</table>

## Manual Circuit Breaker Control

Control the circuit breaker manually when needed:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Reset the circuit breaker to closed state
connectionManager.ResetCircuitBreaker();
```

</td>
<td>

```go
// Reset the circuit breaker to closed state
circuitBreaker.Reset()
```

</td>
</tr>
</table>

This is useful for:
- Administrative intervention after fixing known issues
- Testing and debugging
- Forcing retry after configuration changes

## Exception Handling

Handle circuit breaker exceptions appropriately:

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
    await messagePublisher.PublishAsync(message);
}
catch (CircuitBreakerOpenException ex)
{
    logger.LogWarning("Cannot publish - circuit breaker open");
    // Handle appropriately - queue for later retry
    await localQueue.EnqueueAsync(message);
}
```

</td>
<td>

```go
err := messagePublisher.Publish(ctx, message)
if errors.Is(err, reliability.ErrCircuitBreakerOpen) {
    log.Warn("Cannot publish - circuit breaker open")
    // Handle appropriately - queue for later retry
    return localQueue.Enqueue(ctx, message)
}
```

</td>
</tr>
</table>

## Best Practices

### 1. Choose Appropriate Thresholds

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// For critical services with occasional hiccups
options.CircuitBreakerFailureThreshold = 10;
options.CircuitBreakerOpenTimeoutSeconds = 60;

// For less critical services
options.CircuitBreakerFailureThreshold = 3;
options.CircuitBreakerOpenTimeoutSeconds = 30;
```

</td>
<td>

```go
// For critical services with occasional hiccups
config.CircuitBreaker.FailureThreshold = 10
config.CircuitBreaker.OpenTimeout = 60 * time.Second

// For less critical services
config.CircuitBreaker.FailureThreshold = 3
config.CircuitBreaker.OpenTimeout = 30 * time.Second
```

</td>
</tr>
</table>

### 2. Combine with Retry Policies

The circuit breaker works alongside retry policies:

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
        options.InitialRetryDelayMs = 1000;
        
        // Circuit breaker configuration
        options.EnableCircuitBreaker = true;
        options.CircuitBreakerFailureThreshold = 5;
    });
```

</td>
<td>

```go
config := messaging.Config{
    RabbitMQ: rabbitmq.Config{
        // Retry configuration
        MaxRetryAttempts: 3,
        InitialRetryDelay: time.Second,
        
        // Circuit breaker configuration
        CircuitBreaker: &reliability.CircuitBreakerConfig{
            Enabled: true,
            FailureThreshold: 5,
        },
    },
}
```

</td>
</tr>
</table>

### 3. Handle Open Circuit Gracefully

Implement fallback mechanisms when the circuit is open:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task PublishWithFallback(IMessage message)
{
    try
    {
        await publisher.PublishAsync(message);
    }
    catch (CircuitBreakerOpenException)
    {
        // Fallback options:
        // 1. Store in local queue for later processing
        await localQueue.EnqueueAsync(message);
        
        // 2. Try alternative service
        // await alternativePublisher.PublishAsync(message);
        
        logger.LogWarning(
            "Message queued locally due to circuit breaker");
    }
}
```

</td>
<td>

```go
func (s *Service) PublishWithFallback(
    ctx context.Context, 
    message messaging.Message) error {
    
    err := s.publisher.Publish(ctx, message)
    if errors.Is(err, reliability.ErrCircuitBreakerOpen) {
        // Fallback options:
        // 1. Store in local queue for later processing
        if err := s.localQueue.Enqueue(ctx, message); err != nil {
            return err
        }
        
        // 2. Try alternative service
        // return s.alternativePublisher.Publish(ctx, message)
        
        log.Warn("Message queued locally due to circuit breaker")
        return nil
    }
    
    return err
}
```

</td>
</tr>
</table>

## Testing Circuit Breaker Behavior

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Fact]
public async Task CircuitBreaker_OpensAfterFailureThreshold()
{
    // Configure with low threshold for testing
    var options = new RabbitMqConnectionOptions
    {
        CircuitBreakerFailureThreshold = 2,
        CircuitBreakerOpenTimeoutSeconds = 1
    };
    
    var manager = new RabbitMqConnectionManager(
        logger, Options.Create(options));
    
    // Simulate failures
    for (int i = 0; i < 2; i++)
    {
        await Assert.ThrowsAsync<ConnectionException>(() => 
            manager.GetConnectionAsync());
    }
    
    // Circuit should now be open
    await Assert.ThrowsAsync<CircuitBreakerOpenException>(() => 
        manager.GetConnectionAsync());
}
```

</td>
<td>

```go
func TestCircuitBreaker_OpensAfterFailureThreshold(t *testing.T) {
    // Configure with low threshold for testing
    cb := reliability.NewCircuitBreaker(
        reliability.WithFailureThreshold(2),
        reliability.WithTimeout(time.Second),
    )
    
    // Simulate failures
    for i := 0; i < 2; i++ {
        _, err := cb.Execute(context.Background(), 
            func() (interface{}, error) {
                return nil, errors.New("connection failed")
            })
        assert.Error(t, err)
    }
    
    // Circuit should now be open
    _, err := cb.Execute(context.Background(), 
        func() (interface{}, error) {
            t.Fatal("Should not execute when circuit is open")
            return nil, nil
        })
    
    assert.True(t, errors.Is(err, reliability.ErrCircuitBreakerOpen))
}
```

</td>
</tr>
</table>

## Performance Considerations

The circuit breaker adds minimal overhead:
- State checks are performed in-memory with atomic operations
- No additional network calls or I/O operations
- Failure tracking uses simple counters
- State transitions are logged but don't block operations

## Troubleshooting

### Circuit Breaker Opens Too Frequently

If the circuit breaker opens too often:
1. Increase the failure threshold
2. Check for transient network issues
3. Verify message broker health
4. Review connection timeout settings

### Circuit Breaker Never Recovers

If the circuit breaker stays open:
1. Check if the timeout is too short
2. Verify the service is actually recovered
3. Use manual reset if needed
4. Check for persistent configuration issues