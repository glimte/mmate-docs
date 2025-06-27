# Message Idempotency and Deduplication

## Overview

mmate provides **at-least-once delivery** guarantees at the infrastructure level. This means messages may be delivered multiple times, especially in topic routing scenarios where overlapping patterns create duplicate deliveries. **Idempotency is application responsibility**.

## Topic Routing and Duplicate Delivery

### When Duplicates Occur

**Overlapping Subscriptions**: Multiple patterns can match the same event:

```go
// These patterns overlap for order.created.us-west
subscriber.SubscribeToEventPattern(ctx, "order", "*", "*", ...)     // All orders
subscriber.SubscribeToEventPattern(ctx, "*", "created", "*", ...)   // All created events  
subscriber.SubscribeToEventPattern(ctx, "*", "*", "us-west", ...)   // US West events
```

**Result**: Same service receives 3 copies of `order.created.us-west`.

**Infrastructure Retries**: Network issues, broker restarts, or consumer failures can trigger message redelivery.

### Design Principle

> **Topic routing enables event distribution to multiple interested services. Each service must implement idempotency to handle duplicate deliveries gracefully.**

## Idempotency Strategies

### 1. Message ID Tracking

Use envelope message IDs to detect duplicates:

```go
type OrderService struct {
    processedMessages map[string]bool // In production: use Redis/database
    mu               sync.RWMutex
}

func (s *OrderService) HandleOrderEvent(ctx context.Context, msg contracts.Message) error {
    envelope := msg.GetEnvelope()
    
    s.mu.RLock()
    if s.processedMessages[envelope.MessageID] {
        s.mu.RUnlock()
        log.Printf("Duplicate message %s, skipping", envelope.MessageID)
        return nil // Already processed
    }
    s.mu.RUnlock()
    
    // Process message
    err := s.processOrder(msg)
    if err != nil {
        return err
    }
    
    // Mark as processed
    s.mu.Lock()
    s.processedMessages[envelope.MessageID] = true
    s.mu.Unlock()
    
    return nil
}
```

### 2. Business Logic Idempotency

Use business state to detect duplicates:

```go
func (s *OrderService) CreateOrder(orderEvent *OrderCreatedEvent) error {
    // Check if order already exists
    existing, err := s.repository.GetOrder(orderEvent.OrderID)
    if err != nil && !errors.Is(err, ErrOrderNotFound) {
        return err
    }
    
    if existing != nil {
        log.Printf("Order %s already exists, skipping creation", orderEvent.OrderID)
        return nil // Idempotent - order already created
    }
    
    // Create new order
    return s.repository.CreateOrder(orderEvent.ToOrder())
}
```

### 3. Database Constraints

Use unique constraints to prevent duplicates:

```sql
-- Orders table with unique external ID
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    external_id VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Idempotency log
CREATE TABLE processed_messages (
    message_id VARCHAR(255) PRIMARY KEY,
    processed_at TIMESTAMP DEFAULT NOW(),
    service_name VARCHAR(100) NOT NULL
);
```

```go
func (s *OrderService) CreateOrderIdempotent(orderEvent *OrderCreatedEvent) error {
    tx, err := s.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Try to insert order
    _, err = tx.Exec(
        "INSERT INTO orders (external_id, status) VALUES ($1, $2) ON CONFLICT (external_id) DO NOTHING",
        orderEvent.OrderID, "created",
    )
    if err != nil {
        return err
    }
    
    // Record message processing
    _, err = tx.Exec(
        "INSERT INTO processed_messages (message_id, service_name) VALUES ($1, $2) ON CONFLICT (message_id) DO NOTHING",
        orderEvent.MessageID, "order-service",
    )
    if err != nil {
        return err
    }
    
    return tx.Commit()
}
```

## Production Patterns

### Redis-Based Deduplication

```go
type RedisIdempotencyStore struct {
    client *redis.Client
    ttl    time.Duration
}

func (r *RedisIdempotencyStore) IsProcessed(messageID string) (bool, error) {
    result, err := r.client.Exists(context.Background(), r.key(messageID)).Result()
    return result > 0, err
}

func (r *RedisIdempotencyStore) MarkProcessed(messageID string) error {
    return r.client.Set(context.Background(), r.key(messageID), "1", r.ttl).Err()
}

func (r *RedisIdempotencyStore) key(messageID string) string {
    return fmt.Sprintf("processed:%s", messageID)
}
```

### Distributed Lock Pattern

For operations that must only happen once across multiple instances:

```go
func (s *OrderService) ProcessPayment(paymentEvent *PaymentEvent) error {
    lockKey := fmt.Sprintf("payment-lock:%s", paymentEvent.OrderID)
    
    // Acquire distributed lock
    acquired, err := s.lockStore.AcquireLock(lockKey, 30*time.Second)
    if err != nil {
        return err
    }
    if !acquired {
        log.Printf("Payment %s already being processed", paymentEvent.OrderID)
        return nil // Another instance is processing
    }
    defer s.lockStore.ReleaseLock(lockKey)
    
    // Process payment
    return s.processPaymentInternal(paymentEvent)
}
```

## Topic Routing Best Practices

### 1. Separate Concerns

Design patterns to avoid overlaps:

```go
// Good: Non-overlapping patterns
subscriber.SubscribeToEventPattern(ctx, "order", "*", "*", orderHandler)      // Order service
subscriber.SubscribeToEventPattern(ctx, "payment", "*", "*", paymentHandler)  // Payment service
subscriber.SubscribeToEventPattern(ctx, "inventory", "*", "*", inventoryHandler) // Inventory service
```

### 2. Use Different Services

Instead of multiple patterns in one service:

```go
// Better: Different services for different concerns
// order-service: handles order.*.*
// audit-service: handles *.created.*
// analytics-service: handles *.*.us-west
```

### 3. Correlation ID Tracking

Track related events to avoid processing chains multiple times:

```go
type EventHandler struct {
    processedCorrelations map[string]map[string]bool // correlationID -> messageID -> processed
}

func (h *EventHandler) Handle(msg contracts.Message) error {
    correlationID := msg.GetCorrelationID()
    messageID := msg.GetMessageID()
    
    if h.isCorrelationProcessed(correlationID, messageID) {
        return nil
    }
    
    // Process and mark
    err := h.processEvent(msg)
    if err != nil {
        return err
    }
    
    h.markCorrelationProcessed(correlationID, messageID)
    return nil
}
```

## Load Balancing vs Broadcasting

### When You Want Load Balancing (One Consumer)

Use shared queues:

```go
// Multiple instances share same queue - only one processes each message
subscriber.Subscribe(ctx, "order-processor", "OrderEvent", handler)
```

### When You Want Broadcasting (All Consumers)

Use topic patterns:

```go
// Each service gets its own copy
subscriber.SubscribeToEventPattern(ctx, "order", "*", "*", orderHandler)
subscriber.SubscribeToEventPattern(ctx, "order", "*", "*", auditHandler)  // Different service
```

## Monitoring and Debugging

### Idempotency Metrics

Track duplicate detection:

```go
type IdempotencyMetrics struct {
    duplicatesDetected counter
    messagesProcessed  counter
}

func (h *Handler) Handle(msg contracts.Message) error {
    h.metrics.messagesProcessed.Inc()
    
    if h.isAlreadyProcessed(msg.GetMessageID()) {
        h.metrics.duplicatesDetected.Inc()
        return nil
    }
    
    // Process...
}
```

### Debug Logging

Log idempotency decisions:

```go
func (h *Handler) Handle(msg contracts.Message) error {
    messageID := msg.GetMessageID()
    
    if h.isAlreadyProcessed(messageID) {
        log.Info("Duplicate message detected",
            "messageID", messageID,
            "type", msg.GetType(),
            "correlationID", msg.GetCorrelationID())
        return nil
    }
    
    log.Info("Processing new message",
        "messageID", messageID,
        "type", msg.GetType())
    
    // Process...
}
```

## Summary

- **Infrastructure**: mmate provides at-least-once delivery with topic routing
- **Application**: Must implement idempotency for exactly-once semantics
- **Patterns**: Message ID tracking, business logic checks, database constraints
- **Production**: Use Redis/database for distributed idempotency stores
- **Design**: Prefer non-overlapping patterns or separate services
- **Monitoring**: Track duplicate detection rates and processing metrics

> Remember: Idempotency is not just about preventing errors—it's about building resilient systems that handle the realities of distributed messaging gracefully.