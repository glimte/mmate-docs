# Getting Started with Mmate-Go

This guide will help you get up and running with Mmate-Go in just a few minutes.

## Prerequisites

- Go 1.21 or later
- RabbitMQ 3.8 or later
- Basic knowledge of Go and messaging concepts

## Installation

```bash
go get github.com/mmate/mmate-go
```

## Quick Start

### 1. Start RabbitMQ

Using Docker:
```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
```

Or install locally following the [RabbitMQ installation guide](https://www.rabbitmq.com/download.html).

### 2. Create Your First Publisher

```go
package main

import (
    "context"
    "log"
    "time"
    
    "github.com/mmate/mmate-go/contracts"
    "github.com/mmate/mmate-go/internal/rabbitmq"
    "github.com/mmate/mmate-go/messaging"
)

// Define your message type
type OrderCreatedEvent struct {
    contracts.BaseEvent
    OrderID    string  `json:"orderId"`
    CustomerID string  `json:"customerId"`
    Amount     float64 `json:"amount"`
}

func main() {
    ctx := context.Background()
    
    // Create connection manager
    connManager := rabbitmq.NewConnectionManager("amqp://guest:guest@localhost:5672/")
    if err := connManager.Connect(ctx); err != nil {
        log.Fatal(err)
    }
    defer connManager.Close()
    
    // Create channel pool
    channelPool, err := rabbitmq.NewChannelPool(connManager)
    if err != nil {
        log.Fatal(err)
    }
    defer channelPool.Close()
    
    // Create publisher
    amqpPublisher := rabbitmq.NewPublisher(channelPool)
    publisher := messaging.NewMessagePublisher(amqpPublisher)
    
    // Create and publish event
    event := &OrderCreatedEvent{
        BaseEvent:  contracts.NewBaseEvent("OrderCreatedEvent", "order-123"),
        OrderID:    "order-123",
        CustomerID: "customer-456",
        Amount:     99.99,
    }
    
    err = publisher.PublishEvent(ctx, event)
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("Event published successfully!")
}
```

### 3. Create Your First Consumer

```go
package main

import (
    "context"
    "log"
    "log/slog"
    "os"
    "os/signal"
    "syscall"
    
    "github.com/mmate/mmate-go/contracts"
    "github.com/mmate/mmate-go/internal/rabbitmq"
    "github.com/mmate/mmate-go/messaging"
)

// OrderEventHandler handles order events
type OrderEventHandler struct {
    logger *slog.Logger
}

func (h *OrderEventHandler) Handle(ctx context.Context, msg contracts.Message) error {
    h.logger.Info("Received message",
        "type", msg.GetType(),
        "id", msg.GetID(),
    )
    
    // Process the message here
    // ...
    
    return nil
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
    
    // Create connection manager
    connManager := rabbitmq.NewConnectionManager("amqp://guest:guest@localhost:5672/",
        rabbitmq.WithLogger(logger),
    )
    if err := connManager.Connect(ctx); err != nil {
        log.Fatal(err)
    }
    defer connManager.Close()
    
    // Create channel pool
    channelPool, err := rabbitmq.NewChannelPool(connManager)
    if err != nil {
        log.Fatal(err)
    }
    defer channelPool.Close()
    
    // Create consumer and subscriber
    consumer := rabbitmq.NewConsumer(channelPool)
    dispatcher := messaging.NewMessageDispatcher()
    subscriber := messaging.NewMessageSubscriber(consumer, dispatcher,
        messaging.WithSubscriberLogger(logger),
    )
    
    // Create handler
    handler := &OrderEventHandler{logger: logger}
    
    // Subscribe to events
    err = subscriber.Subscribe(ctx, "evt.order.created", "OrderCreatedEvent", 
        messaging.MessageHandlerFunc(handler.Handle),
        messaging.WithPrefetchCount(10),
    )
    if err != nil {
        log.Fatal(err)
    }
    
    logger.Info("Consumer started, waiting for messages...")
    
    // Wait for interrupt
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan
    
    logger.Info("Shutting down...")
}
```

## Common Patterns

### 1. Request-Reply Pattern

```go
// Using the bridge for synchronous request-response
bridge, err := bridge.NewSyncAsyncBridge(publisher, subscriber, logger)
if err != nil {
    log.Fatal(err)
}

// Send command and wait for reply
command := &ProcessOrderCommand{
    BaseCommand: contracts.NewBaseCommand("ProcessOrderCommand"),
    OrderID:     "order-123",
}

reply, err := bridge.SendAndWait(ctx, command, "cmd.order.process", 30*time.Second)
if err != nil {
    log.Printf("Request failed: %v", err)
    return
}

log.Printf("Reply received: %+v", reply)
```

### 2. Event Sourcing Pattern

```go
// Publish events in order
events := []contracts.Event{
    &OrderCreatedEvent{...},
    &OrderItemAddedEvent{...},
    &OrderConfirmedEvent{...},
}

for _, event := range events {
    if err := publisher.PublishEvent(ctx, event); err != nil {
        log.Printf("Failed to publish event: %v", err)
        return
    }
}
```

### 3. Work Queue Pattern

```go
// Create consumer group for load balancing
group := messaging.NewConsumerGroup("order-processors",
    messaging.WithGroupSize(5),
    messaging.WithMaxWorkers(10),
)

// All consumers in the group share the workload
err = group.Subscribe(ctx, "cmd.order.process", handler)
```

## Configuration Options

### Connection Options
```go
connManager := rabbitmq.NewConnectionManager(amqpURL,
    rabbitmq.WithLogger(logger),
    rabbitmq.WithReconnectDelay(5*time.Second),
    rabbitmq.WithHeartbeat(30*time.Second),
)
```

### Publisher Options
```go
publisher := rabbitmq.NewPublisher(channelPool,
    rabbitmq.WithConfirmMode(true),
    rabbitmq.WithPublisherLogger(logger),
)
```

### Subscriber Options
```go
subscriber.Subscribe(ctx, queue, messageType, handler,
    messaging.WithPrefetchCount(10),
    messaging.WithAutoAck(false),
    messaging.WithDeadLetterExchange("dlx"),
)
```

## Error Handling

### Retry with Exponential Backoff
```go
retryPolicy := reliability.NewExponentialBackoff(
    100*time.Millisecond, // initial delay
    5*time.Second,        // max delay
    2.0,                  // multiplier
    3,                    // max attempts
)

// Use with bridge
bridge, err := bridge.NewSyncAsyncBridge(publisher, subscriber, logger,
    bridge.WithRetryPolicy(retryPolicy),
)
```

### Circuit Breaker
```go
breaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithTimeout(30*time.Second),
)

// Wrap operations
err := breaker.Execute(ctx, func() error {
    return publisher.PublishEvent(ctx, event)
})
```

## Monitoring

### Health Checks
```go
registry := health.NewRegistry()
registry.Register(health.NewRabbitMQChecker(connManager, logger))
registry.Register(health.NewQueueChecker("my-queue", channelPool, logger))

// Check health
status := registry.Check(ctx)
if status.Status != health.StatusHealthy {
    log.Printf("System unhealthy: %v", status)
}
```

### Metrics
```go
// Use interceptors for metrics
chain := interceptors.NewChain(
    interceptors.Metrics(metricsCollector),
    interceptors.Logging(logger),
)

publisher = publisher.WithInterceptor(chain)
```

## Testing

### Unit Testing
```go
func TestOrderHandler(t *testing.T) {
    handler := &OrderEventHandler{logger: slog.Default()}
    
    event := &OrderCreatedEvent{
        BaseEvent:  contracts.NewBaseEvent("OrderCreatedEvent", "order-123"),
        OrderID:    "order-123",
        CustomerID: "customer-456",
        Amount:     99.99,
    }
    
    err := handler.Handle(context.Background(), event)
    assert.NoError(t, err)
}
```

### Integration Testing
```go
func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }
    
    // Setup test RabbitMQ
    // ... test implementation
}
```

## Next Steps

Now that you have the basics:

1. Learn about [Message Patterns](patterns.md) for advanced scenarios
2. Explore [StageFlow](stageflow.md) for workflow orchestration
3. Read about [Interceptors](interceptors.md) for cross-cutting concerns
4. Check the [API Reference](api/core.md) for detailed documentation

## Troubleshooting

### Connection Issues
- Verify RabbitMQ is running: `curl -i http://localhost:15672/api/overview`
- Check credentials and permissions
- Ensure firewall allows connections

### Message Not Received
- Verify queue exists and has messages
- Check queue bindings
- Ensure message type matches handler registration

### Performance Issues
- Increase prefetch count for higher throughput
- Use channel pooling for concurrent operations
- Consider consumer groups for scaling

For more help, see the [Troubleshooting Guide](troubleshooting.md).