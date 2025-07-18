# Getting Started with Mmate-Go

This guide will help you get up and running with Mmate-Go in just a few minutes.

## Important: API Changes

**Note**: The mmate-go library provides a simplified public API through the main package. You should use `mmate.NewClient()` to create clients instead of directly accessing internal packages.

### Key Features:
- **Automatic Queue Creation**: Queues are automatically created when you subscribe to them
- **Clean Public API**: No need to import or use internal packages
- **Transport Abstraction**: Designed to support multiple transports (RabbitMQ, Kafka, SQS) in the future
- **Simplified Setup**: Single client creation with configuration
- **Message Type Registration**: Register your message types for automatic deserialization
- **Enterprise Features**: TTL retry scheduling, acknowledgment tracking, sync mutation journal

## Prerequisites

- Go 1.21 or later
- RabbitMQ 3.8 or later
- Basic knowledge of Go and messaging concepts

## Installation

```bash
go get github.com/glimte/mmate-go
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

### 2. Message Type Registration

Before using custom message types, you must register them with the messaging framework. This allows the framework to properly deserialize messages from the wire format:

```go
import "github.com/glimte/mmate-go/messaging"

// Register your message types at startup
messaging.Register("OrderCreatedEvent", func() contracts.Message {
    return &OrderCreatedEvent{}
})

messaging.Register("ProcessOrderCommand", func() contracts.Message {
    return &ProcessOrderCommand{}
})
```

**Important**: Message type names must match exactly between publisher and consumer. The type name is used for routing and deserialization.

### 3. Create Your First Publisher

```go
package main

import (
    "context"
    "log"
    "time"
    
    mmate "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/contracts"
    "github.com/glimte/mmate-go/messaging"
)

// Define your message type
type OrderCreatedEvent struct {
    contracts.BaseMessage
    OrderID    string  `json:"orderId"`
    CustomerID string  `json:"customerId"`
    Amount     float64 `json:"amount"`
}

func main() {
    ctx := context.Background()
    
    // Register message types for proper deserialization
    messaging.Register("OrderCreatedEvent", func() contracts.Message {
        return &OrderCreatedEvent{}
    })
    
    // Create mmate client with auto-queue creation and service configuration
    client, err := mmate.NewClientWithOptions("amqp://admin:admin@localhost:5672/",
        mmate.WithServiceName("order-publisher"),
        mmate.WithQueueBindings(
            messaging.QueueBinding{
                Exchange:   "mmate.events",
                RoutingKey: "order.publisher.*",
            },
        ),
    )
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    defer client.Close()
    
    log.Printf("✅ Connected! Service queue: %s", client.ServiceQueue())
    
    // Get the publisher
    publisher := client.Publisher()
    
    // Create and publish event
    event := &OrderCreatedEvent{}
    event.Type = "OrderCreatedEvent"
    event.ID = "order-123"
    event.CorrelationID = "corr-" + time.Now().Format("20060102150405")
    event.Timestamp = time.Now()
    event.OrderID = "order-123"
    event.CustomerID = "customer-456"
    event.Amount = 99.99
    
    err = publisher.Publish(ctx, event,
        messaging.WithExchange("mmate.events"),
        messaging.WithRoutingKey("order.service.created"),
        messaging.WithPersistent(true),
    )
    if err != nil {
        log.Fatalf("Failed to publish: %v", err)
    }
    
    log.Println("✅ Event published successfully!")
}
```

### 4. Create Your First Consumer

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "syscall"
    
    mmate "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/contracts"
    "github.com/glimte/mmate-go/messaging"
)

// Define the message type to match the publisher
type OrderCreatedEvent struct {
    contracts.BaseMessage
    OrderID    string  `json:"orderId"`
    CustomerID string  `json:"customerId"`
    Amount     float64 `json:"amount"`
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Register message types for proper deserialization
    messaging.Register("OrderCreatedEvent", func() contracts.Message {
        return &OrderCreatedEvent{}
    })
    
    // Create mmate client with auto-queue creation
    client, err := mmate.NewClientWithOptions("amqp://admin:admin@localhost:5672/",
        mmate.WithServiceName("order-processor"),
        mmate.WithQueueBindings(
            messaging.QueueBinding{
                Exchange:   "mmate.events",
                RoutingKey: "order.service.*", // Listen to order service events
            },
        ),
    )
    if err != nil {
        log.Fatalf("Failed to create client: %v", err)
    }
    defer client.Close()
    
    log.Printf("✅ Connected! Service queue: %s", client.ServiceQueue())
    
    // Get subscriber and dispatcher
    subscriber := client.Subscriber()
    dispatcher := client.Dispatcher()
    
    // Register handler
    handler := messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
        event, ok := msg.(*OrderCreatedEvent)
        if !ok {
            log.Printf("❌ Unexpected message type: %T", msg)
            return nil
        }
        
        log.Printf("📦 Processing order: OrderID=%s, CustomerID=%s, Amount=%.2f", 
            event.OrderID, event.CustomerID, event.Amount)
        
        // Process the order here
        // ... business logic ...
        
        return nil // Auto-ack on success
    })
    
    err = dispatcher.RegisterHandler(&OrderCreatedEvent{}, handler)
    if err != nil {
        log.Fatalf("Failed to register handler: %v", err)
    }
    
    // Subscribe to the auto-created service queue
    err = subscriber.Subscribe(ctx, client.ServiceQueue(), "OrderCreatedEvent", 
        dispatcher, 
        messaging.WithAutoAck(true),        // Enable auto-acknowledgment
        messaging.WithPrefetchCount(10),    // Process 10 messages concurrently
    )
    if err != nil {
        log.Fatalf("Failed to subscribe: %v", err)
    }
    
    log.Printf("✅ Consumer started, waiting for messages on: %s", client.ServiceQueue())
    
    // Wait for interrupt
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan
    
    log.Println("🛑 Shutting down...")
}
```

## Common Patterns

### 1. Request-Reply Pattern

```go
// The bridge is already integrated in the client
// Send command and wait for reply
command := &ProcessOrderCommand{
    BaseCommand: contracts.BaseCommand{
        BaseMessage: contracts.NewBaseMessage("ProcessOrderCommand"),
    },
    OrderID: "order-123",
}

reply, err := client.Bridge().SendAndWait(ctx, command, "cmd.order.process", 30*time.Second)
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

publisher := client.Publisher()
for _, event := range events {
    err := publisher.Publish(ctx, event,
        messaging.WithExchange("mmate.events"),
        messaging.WithRoutingKey("event.order."+event.GetType()))
    if err != nil {
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

### Client Options
```go
// Configure the client with various options
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithLogger(logger),
    mmate.WithFIFOMode(true),
    mmate.WithQueueBindings(
        messaging.QueueBinding{
            Exchange:   "mmate.events",
            RoutingKey: "orders.*",
        },
    ),
)

// Enable FIFO (First-In-First-Out) message ordering
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithFIFOMode(true), // Ensures message ordering
)

// Service queue is automatically created as "{serviceName}-queue"
// For example, if service name is "order-processor", 
// the queue will be "order-processor-queue"
fmt.Printf("Service queue: %s\n", client.ServiceQueue())
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
// Configure retry policy at client level
retryPolicy := reliability.NewExponentialBackoff(
    100*time.Millisecond, // initial delay
    5*time.Second,        // max delay
    2.0,                  // multiplier
    3,                    // max attempts
)

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithRetryPolicy(retryPolicy),
)

// Or use default retry configuration
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithDefaultRetry(),
)
```

### Circuit Breaker
```go
// Configure circuit breaker at client level
circuitBreaker := reliability.NewCircuitBreaker(
    reliability.WithFailureThreshold(5),
    reliability.WithSuccessThreshold(2),
    reliability.WithTimeout(30*time.Second),
)

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithCircuitBreaker(circuitBreaker),
)
```

## Monitoring

### Health Checks
```go
// Service-scoped health monitoring
health, err := client.GetServiceHealth(ctx)
if err != nil {
    log.Printf("Health check failed: %v", err)
}

// Get service metrics
metrics, err := client.GetServiceMetrics(ctx)
summary := client.GetMetricsSummary()

// Create service monitor for advanced monitoring
monitor, err := client.NewServiceMonitor()
if err != nil {
    log.Printf("Failed to create monitor: %v", err)
}
```

### Metrics
```go
// Use default metrics
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithDefaultMetrics(),
)

// Or provide custom metrics collector
customCollector := &MyMetricsCollector{}
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithMetrics(customCollector),
)
```

## Interceptors

### Configure Interceptor Pipeline
```go
// Create interceptor pipeline
pipeline := interceptors.NewPipeline()
pipeline.Add(interceptors.NewLoggingInterceptor(logger))
pipeline.Add(interceptors.NewMetricsInterceptor(metrics))
pipeline.Add(interceptors.NewValidationInterceptor())

// Configure client with interceptors
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithInterceptors(pipeline),
)

// Or configure separate pipelines for publish/subscribe
publishPipeline := interceptors.NewPipeline().
    Add(interceptors.NewLoggingInterceptor(logger))

subscribePipeline := interceptors.NewPipeline().
    Add(interceptors.NewValidationInterceptor()).
    Add(interceptors.NewMetricsInterceptor(metrics))

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithPublishInterceptors(publishPipeline),
    mmate.WithSubscribeInterceptors(subscribePipeline),
)
```

## Dead Letter Queue (DLQ) Handling

```go
// Configure DLQ handler
dlqHandler := reliability.NewDLQHandler(
    reliability.WithMaxRetries(3),
    reliability.WithDLQExchange("mmate.dlx"),
    reliability.WithRetryDelay(5*time.Second),
)

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("my-service"),
    mmate.WithDLQHandler(dlqHandler),
)
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

## Enterprise Features

Mmate-Go includes enterprise-grade features for production deployments:

### TTL-based Retry Scheduling

Persistent retry scheduling using RabbitMQ Dead Letter Exchange (DLX):

```go
// Enable TTL retry with custom policy
retryPolicy := reliability.NewExponentialBackoff(
    100*time.Millisecond,  // initial delay
    30*time.Second,        // max delay
    2.0,                   // multiplier
    5,                     // max retries
)

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithTTLRetry(retryPolicy),  // Enable TTL retry
    mmate.WithServiceName("order-service"),
)

// Retries persist across service restarts
```

### Application-Level Acknowledgment Tracking

End-to-end message processing visibility:

```go
// Enable acknowledgment tracking
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithAcknowledgmentTracking(30*time.Second),  // timeout
    mmate.WithServiceName("order-service"),
)

// Send message and wait for processing confirmation
ackResponse, err := client.SendWithAck(ctx, command)
if err != nil {
    log.Fatal("Failed to send:", err)
}

// Wait for acknowledgment
ack, err := ackResponse.WaitForAcknowledgment(ctx)
if err != nil {
    log.Printf("Acknowledgment timeout: %v", err)
} else if ack.Success {
    log.Printf("Message processed in %v", ack.ProcessingTime)
} else {
    log.Printf("Processing failed: %s", ack.ErrorMessage)
}
```

### Sync Mutation Journal

Entity-level mutation tracking for distributed synchronization:

```go
// Enable sync mutation journal
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithSyncMutationJournal(),
    mmate.WithServiceName("order-service"),
)

// Record entity mutations
mutation := &journal.EntityMutationRecord{
    EntityType:    "Order",
    EntityID:      order.ID,
    EntityVersion: order.Version,
    MutationType:  journal.EntityUpdate,
    CorrelationID: correlationID,
    AfterState:    orderData,
    SyncStatus:    journal.SyncStatusPending,
}

err = client.RecordEntityMutation(ctx, mutation)

// Query entity history
mutations, err := client.GetEntityMutations(ctx, "Order", orderID)

// Get unsynced mutations for background sync
unsyncedMutations, err := client.GetUnsyncedMutations(ctx, 100)
```

### Combined Enterprise Configuration

Enable all enterprise features together:

```go
client, err := mmate.NewClientWithOptions(connectionString,
    // Enterprise features
    mmate.WithTTLRetry(),                           // Persistent retry
    mmate.WithAcknowledgmentTracking(30*time.Second), // Ack tracking
    mmate.WithSyncMutationJournal(),                  // Mutation journal
    
    // Standard configuration
    mmate.WithServiceName("order-service"),
    mmate.WithDefaultMetrics(),
)
```

## Next Steps

Now that you have the basics:

1. Learn about [FIFO Queues](../components/messaging.md#fifo-queues) for ordered processing
2. Explore [Message Patterns](../patterns.md) for advanced scenarios  
3. Read about [StageFlow](../components/stageflow.md) for workflow orchestration
4. Check [Interceptors](../components/interceptors.md) for cross-cutting concerns
5. Explore [Enterprise Features](../advanced/) for production deployments:
   - [TTL Retry Scheduler](../advanced/ttl-retry-scheduler.md)
   - [Acknowledgment Tracking](../advanced/acknowledgment-tracking.md)
   - [Sync Mutation Journal](../advanced/sync-mutation-journal.md)
6. See the [API Reference](../platform/go/api-reference.md) for detailed documentation

## Troubleshooting

### Connection Issues
- Verify RabbitMQ is running: `curl -i http://localhost:15672/api/overview`
- Check credentials and permissions
- Ensure firewall allows connections

### Message Not Received
- Verify queue exists and has messages
- Check queue bindings
- Ensure message type matches handler registration
- **Verify message types are registered** with `messaging.Register()` before creating the client
- Check that message type names match exactly between publisher and consumer

### Performance Issues
- Increase prefetch count for higher throughput
- Use channel pooling for concurrent operations
- Consider consumer groups for scaling

For more help, see the [Advanced Topics](../advanced/README.md) section.