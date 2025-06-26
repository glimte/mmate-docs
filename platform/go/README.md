# Go Platform Documentation

This section contains Go-specific documentation for Mmate.

## Contents

- [API Reference](api-reference.md) - Complete Go API documentation
- [Configuration](configuration.md) - Go-specific configuration options
- [Examples](examples.md) - Go code examples and patterns

## Quick Links

- [Getting Started Guide](../../getting-started/go.md)
- [Go Repository](https://github.com/mmate/mmate-go)

## Go-Specific Features

### Manual Dependency Injection
Unlike .NET's built-in DI, Go uses explicit wiring:

```go
connManager := rabbitmq.NewConnectionManager(url)
channelPool, _ := rabbitmq.NewChannelPool(connManager)
publisher := messaging.NewMessagePublisher(
    rabbitmq.NewPublisher(channelPool))
```

### Context-Based Cancellation
All operations accept a context for cancellation and timeout:

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

err := publisher.PublishEvent(ctx, event)
```

### Error Handling
Go uses explicit error returns instead of exceptions:

```go
reply, err := bridge.SendAndWait(ctx, query, "routing.key", timeout)
if err != nil {
    var timeoutErr *TimeoutError
    if errors.As(err, &timeoutErr) {
        // Handle timeout
    }
    return nil, err
}
```

### Goroutines for Concurrency
Use goroutines instead of async/await:

```go
var wg sync.WaitGroup
for i := 0; i < workers; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        // Worker logic
    }()
}
wg.Wait()
```

## Common Patterns

### Configuration
Environment variables with struct tags:

```go
type Config struct {
    ConnectionString string `env:"MMATE_CONNECTION_STRING"`
    PrefetchCount    int    `env:"MMATE_PREFETCH_COUNT" default:"10"`
}
```

### Testing
Table-driven tests and mocks:

```go
func TestHandler(t *testing.T) {
    tests := []struct {
        name    string
        input   Message
        wantErr bool
    }{
        // Test cases
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Test logic
        })
    }
}
```

### Logging
Structured logging with slog:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("Processing message",
    "messageId", msg.GetID(),
    "type", msg.GetType())
```

## Best Practices

1. **Always handle errors** - Don't ignore error returns
2. **Use contexts** - Pass context through your call chain
3. **Defer cleanup** - Use defer for resource cleanup
4. **Avoid goroutine leaks** - Ensure goroutines can exit
5. **Use interfaces** - Define small, focused interfaces