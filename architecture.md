# Mmate Architecture Overview

This document describes the architecture of Mmate, a messaging framework available for both .NET and Go. While implementation details differ between platforms, the core concepts and wire format remain consistent.

## Core Concepts

### 1. Message-Driven Architecture

Mmate implements several messaging patterns:
- **Publish/Subscribe** - Event broadcasting to multiple consumers
- **Request/Reply** - Synchronous-style communication over async transport
- **Work Queues** - Load-balanced task distribution
- **Topic Routing** - Pattern-based message routing
- **Saga/Workflow** - Multi-step orchestrated processes

### 2. Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                      │
├─────────────────────────────────────────────────────────────┤
│  StageFlow  │  Bridge  │  Interceptors  │  Schema/Contracts │
├─────────────────────────────────────────────────────────────┤
│                    Messaging Core                           │
│  Publisher  │  Subscriber  │  Dispatcher  │  Handlers       │
├─────────────────────────────────────────────────────────────┤
│                  Discovery & Registry                       │
│  Contract Discovery  │  Endpoint Registry  │  Announcements │
├─────────────────────────────────────────────────────────────┤
│                    Reliability Layer                        │
│  Retry     │  Circuit Breaker  │  DLQ    │  Monitoring      │
├─────────────────────────────────────────────────────────────┤
│                    Transport Adapter                        │
│                      RabbitMQ/AMQP                          │
└─────────────────────────────────────────────────────────────┘
```

### 3. Message Flow

#### Publishing Flow
1. Application creates a typed message
2. Message passes through interceptor chain
3. Message is serialized to JSON envelope
4. Published to RabbitMQ exchange with routing key
5. RabbitMQ routes to appropriate queues

#### Consumption Flow
1. Consumer receives message from queue
2. Message envelope is deserialized
3. Type-specific message is reconstructed
4. Message passes through interceptor chain
5. Dispatcher routes to appropriate handler
6. Handler processes message
7. Acknowledgment sent to broker

## Platform-Specific Architecture

### .NET Implementation

```csharp
// Dependency Injection based
services.AddMmate(options =>
{
    options.ConnectionString = "amqp://localhost";
    options.AddInterceptor<LoggingInterceptor>();
});

// Async/await pattern
public async Task<TReply> SendAndWaitAsync<TReply>(ICommand command)
{
    return await _bridge.SendAndWaitAsync<TReply>(command);
}
```

**Key Characteristics:**
- Uses ASP.NET Core dependency injection
- async/await for asynchronous operations
- Attribute-based handler discovery (removed in recent versions)
- Strong typing with generics
- Configuration via appsettings.json

### Go Implementation

```go
// Manual wiring
connManager := rabbitmq.NewConnectionManager(amqpURL)
channelPool, _ := rabbitmq.NewChannelPool(connManager)
publisher := messaging.NewMessagePublisher(rabbitmq.NewPublisher(channelPool))

// Context-based cancellation
func (s *Service) SendAndWait(ctx context.Context, cmd contracts.Command) (contracts.Reply, error) {
    return s.bridge.SendAndWait(ctx, cmd, "routing.key", 30*time.Second)
}
```

**Key Characteristics:**
- Explicit dependency wiring
- Context for cancellation and timeouts
- Interface-based design
- Error returns instead of exceptions
- Configuration via environment variables

## Cross-Platform Compatibility

### Wire Format

Both implementations use the same JSON envelope format:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "OrderCreatedEvent",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "req-123",
  "payload": {
    "orderId": "ORDER-456",
    "customerId": "CUST-789",
    "amount": 99.99
  }
}
```

### Queue Naming Conventions

Consistent across both platforms:
- Commands: `cmd.{service}.{command}`
- Events: `evt.{aggregate}.{event}`
- Replies: `rpl.{correlationId}`
- Dead Letter: `dlq.{originalQueue}`
- StageFlow: `stageflow.{flowId}.stage{index}`

## Component Details

### 1. Contracts
- Define message types (Command, Event, Query, Reply)
- Ensure type safety
- Provide serialization hints

### 2. Messaging Core
- **Publisher**: Sends messages to exchanges
- **Subscriber**: Consumes messages from queues
- **Dispatcher**: Routes messages to handlers
- **Handlers**: Process specific message types

### 3. StageFlow
- Multi-stage workflow orchestration
- Compensation support for rollback
- State persistence between stages
- Distributed execution capability

### 4. Bridge (Go) / SyncAsyncBridge (.NET)
- Enables request/reply pattern
- Correlation of requests and responses
- Timeout handling
- Synchronous API over async transport

### 5. Interceptors
- Cross-cutting concerns (logging, metrics, tracing)
- Pre and post-processing hooks
- Chain of responsibility pattern
- Platform-specific implementations

### 6. Contract Discovery
- **Endpoint Registry**: Services register their available endpoints
- **Discovery Protocol**: Query endpoints by ID or pattern
- **Announcements**: Periodic broadcasts of available contracts
- **Decentralized**: Each service owns its contracts
- **Schema Integration**: Contracts include JSON schemas

### 7. Reliability
- **Retry Policies**: Exponential backoff, fixed delay
- **Circuit Breaker**: Prevent cascading failures
- **Dead Letter Queue**: Handle poison messages
- **Health Monitoring**: System health checks

## Deployment Architecture

### Single Service
```
┌─────────────┐
│   Service   │
│   + Mmate   │
└──────┬──────┘
       │
┌──────┴──────┐
│  RabbitMQ   │
└─────────────┘
```

### Microservices
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Service A  │     │  Service B  │     │  Service C  │
│   + Mmate   │     │   + Mmate   │     │   + Mmate   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                   ┌───────┴────────┐
                   │    RabbitMQ    │
                   │    Cluster     │
                   └────────────────┘
```

### High Availability
- Multiple RabbitMQ nodes with clustering
- Queue mirroring for redundancy
- Connection failover
- Persistent messages for durability

## Performance Characteristics

### .NET
- JIT compilation overhead initially
- Excellent async I/O performance
- Higher memory footprint
- Good for CPU-intensive operations

### Go
- Compiled binary, fast startup
- Lightweight goroutines
- Lower memory footprint
- Excellent for high-concurrency scenarios

## Security Considerations

Both platforms support:
- TLS for transport encryption
- Username/password authentication
- Certificate-based authentication
- Message-level encryption (application layer)
- Queue-level access control

## Monitoring and Observability

### Health Checks
- Connection health
- Channel availability
- Queue depth monitoring
- Consumer lag tracking

### Metrics
- Message throughput
- Processing latency
- Error rates
- Resource utilization

### Distributed Tracing
- Correlation ID propagation
- Integration with:
  - .NET: Application Insights, OpenTelemetry
  - Go: OpenTelemetry, Jaeger

## Best Practices

1. **Message Design**
   - Keep messages small and focused
   - Include all necessary data
   - Version messages for compatibility
   - Use correlation IDs for tracing

2. **Error Handling**
   - Implement proper retry strategies
   - Use circuit breakers for external calls
   - Handle poison messages with DLQ
   - Log errors with context

3. **Performance**
   - Use appropriate prefetch counts
   - Batch operations when possible
   - Monitor queue depths
   - Scale consumers based on load

4. **Deployment**
   - Use connection pooling
   - Implement health checks
   - Monitor resource usage
   - Plan for message persistence

## Next Steps

- [Getting Started - .NET](getting-started/dotnet.md)
- [Getting Started - Go](getting-started/go.md)
- [Platform-Specific Guides](README.md#platform-specific)
- [Message Patterns](patterns.md)
- [Component Documentation](components/README.md)