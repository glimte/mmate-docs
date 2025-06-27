# Go Examples

This page contains practical examples of using Mmate in Go applications.

## Basic Examples

### Hello World Publisher

```go
package main

import (
    "context"
    "log"
    mmate "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/contracts"
)

// Define a simple event
type HelloEvent struct {
    contracts.BaseEvent
    Message string `json:"message"`
    From    string `json:"from"`
}

func main() {
    ctx := context.Background()
    
    // Create client
    client, err := mmate.NewClient("amqp://localhost")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
    
    // Get publisher
    publisher := client.Publisher()
    
    // Publish event
    event := &HelloEvent{
        BaseEvent: contracts.NewBaseEvent("HelloEvent", "hello-123"),
        Message:   "Hello, Mmate!",
        From:      "example-app",
    }
    
    err = publisher.PublishEvent(ctx, event)
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("Event published successfully")
}
```

### Hello World Consumer

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    mmate "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/contracts"
    "github.com/glimte/mmate-go/messaging"
)

type HelloEvent struct {
    contracts.BaseEvent
    Message string `json:"message"`
    From    string `json:"from"`
}

type HelloHandler struct{}

func (h *HelloHandler) Handle(ctx context.Context, msg contracts.Message) error {
    event := msg.(*HelloEvent)
    log.Printf("Received: %s from %s\n", event.Message, event.From)
    return nil
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Handle shutdown
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt)
    
    // Create client
    client, err := mmate.NewClient("amqp://localhost")
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
    
    // Get subscriber and dispatcher
    subscriber := client.Subscriber()
    dispatcher := client.Dispatcher()
    
    // Register handler
    dispatcher.RegisterHandler("HelloEvent", 
        messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
            return (&HelloHandler{}).Handle(ctx, msg)
        }))
    
    // Subscribe
    err = subscriber.Subscribe(ctx, "hello.queue", "HelloEvent", 
        dispatcher, messaging.WithPrefetchCount(10))
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("Listening for messages...")
    <-sigChan
    log.Println("Shutting down...")
}
```

## Request/Reply Pattern

### Service Implementation

```go
package main

import (
    "context"
    "fmt"
    "github.com/glimte/mmate-go/contracts"
    "github.com/glimte/mmate-go/messaging"
)

// Query definition
type GetProductQuery struct {
    contracts.BaseQuery
    ProductID string `json:"productId"`
}

// Reply definition
type ProductReply struct {
    contracts.BaseReply
    Product *Product `json:"product,omitempty"`
    Error   string   `json:"error,omitempty"`
}

type Product struct {
    ID          string  `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    InStock     bool    `json:"inStock"`
}

// Handler
type ProductHandler struct {
    catalog   ProductCatalog
    publisher messaging.Publisher
}

func (h *ProductHandler) Handle(ctx context.Context, msg contracts.Message) error {
    query := msg.(*GetProductQuery)
    
    product, err := h.catalog.GetProduct(ctx, query.ProductID)
    
    reply := &ProductReply{
        BaseReply: contracts.NewBaseReply(
            query.GetID(), query.GetCorrelationID()),
    }
    
    if err != nil {
        reply.Success = false
        reply.Error = err.Error()
    } else {
        reply.Success = true
        reply.Product = product
    }
    
    return h.publisher.PublishReply(ctx, reply, query.ReplyTo)
}

// Client using bridge
func GetProduct(ctx context.Context, bridge *bridge.Bridge, 
    productID string) (*Product, error) {
    
    query := &GetProductQuery{
        BaseQuery: contracts.NewBaseQuery("GetProductQuery"),
        ProductID: productID,
    }
    
    reply, err := bridge.SendAndWait(ctx, query, 
        "qry.catalog.products", 10*time.Second)
    if err != nil {
        return nil, err
    }
    
    productReply := reply.(*ProductReply)
    if !productReply.Success {
        return nil, fmt.Errorf("query failed: %s", productReply.Error)
    }
    
    return productReply.Product, nil
}
```

## Command Handler with Validation

```go
package main

import (
    "context"
    "errors"
    "github.com/glimte/mmate-go/contracts"
)

type CreateOrderCommand struct {
    contracts.BaseCommand
    CustomerID  string      `json:"customerId"`
    Items       []OrderItem `json:"items"`
    ShippingAddress Address `json:"shippingAddress"`
}

type OrderItem struct {
    ProductID string  `json:"productId"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

func (c *CreateOrderCommand) Validate() error {
    if c.CustomerID == "" {
        return errors.New("customer ID is required")
    }
    
    if len(c.Items) == 0 {
        return errors.New("order must have at least one item")
    }
    
    for i, item := range c.Items {
        if item.ProductID == "" {
            return fmt.Errorf("item %d: product ID is required", i)
        }
        if item.Quantity <= 0 {
            return fmt.Errorf("item %d: quantity must be positive", i)
        }
        if item.Price < 0 {
            return fmt.Errorf("item %d: price cannot be negative", i)
        }
    }
    
    return c.ShippingAddress.Validate()
}

type OrderHandler struct {
    orderService OrderService
    publisher    messaging.Publisher
}

func (h *OrderHandler) Handle(ctx context.Context, msg contracts.Message) error {
    cmd := msg.(*CreateOrderCommand)
    
    // Validate command
    if err := cmd.Validate(); err != nil {
        return messaging.NewPermanentError("validation failed", err)
    }
    
    // Create order
    orderID, err := h.orderService.CreateOrder(ctx, cmd)
    if err != nil {
        return fmt.Errorf("failed to create order: %w", err)
    }
    
    // Publish event
    event := &OrderCreatedEvent{
        BaseEvent:  contracts.NewBaseEvent("OrderCreatedEvent", orderID),
        OrderID:    orderID,
        CustomerID: cmd.CustomerID,
        TotalAmount: calculateTotal(cmd.Items),
        CreatedAt:  time.Now(),
    }
    
    return h.publisher.PublishEvent(ctx, event)
}
```

## StageFlow Workflow

```go
package main

import (
    "context"
    "github.com/glimte/mmate-go/stageflow"
)

// Workflow context
type OrderFulfillmentContext struct {
    OrderID    string
    CustomerID string
    Items      []OrderItem
    
    // Stage results
    PaymentID       string
    InventoryReserved bool
    ShipmentID      string
    TrackingNumber  string
}

// Stage implementations
type ValidateOrderStage struct{}

func (s *ValidateOrderStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    if wfCtx.OrderID == "" {
        return errors.New("order ID is required")
    }
    if len(wfCtx.Items) == 0 {
        return errors.New("order has no items")
    }
    return nil
}

type ProcessPaymentStage struct {
    paymentService PaymentService
}

func (s *ProcessPaymentStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    paymentID, err := s.paymentService.ProcessPayment(ctx, PaymentRequest{
        OrderID:    wfCtx.OrderID,
        CustomerID: wfCtx.CustomerID,
        Amount:     calculateTotal(wfCtx.Items),
    })
    if err != nil {
        return err
    }
    
    wfCtx.PaymentID = paymentID
    return nil
}

type RefundPaymentStage struct {
    paymentService PaymentService
}

func (s *RefundPaymentStage) Compensate(ctx context.Context, 
    wfCtx *OrderFulfillmentContext, stageErr error) error {
    if wfCtx.PaymentID != "" {
        return s.paymentService.RefundPayment(ctx, wfCtx.PaymentID)
    }
    return nil
}

// Create and execute workflow
func ProcessOrder(ctx context.Context, orderID string) error {
    // Create workflow
    workflow := stageflow.NewFlow[*OrderFulfillmentContext](
        "order-fulfillment",
        stageflow.WithTimeout(10*time.Minute))
    
    // Add stages
    workflow.AddStage("validate", &ValidateOrderStage{})
    
    workflow.AddStage("payment", &ProcessPaymentStage{
        paymentService: paymentService,
    })
    workflow.AddCompensation("payment", &RefundPaymentStage{
        paymentService: paymentService,
    })
    
    workflow.AddStage("inventory", &ReserveInventoryStage{
        inventory: inventoryService,
    })
    workflow.AddCompensation("inventory", &ReleaseInventoryStage{
        inventory: inventoryService,
    })
    
    workflow.AddStage("shipping", &CreateShipmentStage{
        shipping: shippingService,
    })
    
    // Execute
    context := &OrderFulfillmentContext{
        OrderID:    orderID,
        CustomerID: order.CustomerID,
        Items:      order.Items,
    }
    
    result, err := workflow.Execute(ctx, context)
    if err != nil {
        log.Printf("Workflow failed: %v", err)
        return err
    }
    
    log.Printf("Order processed successfully: %+v", result)
    return nil
}
```

## Interceptor Example

```go
package main

import (
    "context"
    "time"
    "github.com/glimte/mmate-go/interceptors"
    "github.com/prometheus/client_golang/prometheus"
)

// Metrics interceptor
type MetricsInterceptor struct {
    messagesProcessed *prometheus.CounterVec
    processingDuration *prometheus.HistogramVec
}

func NewMetricsInterceptor() *MetricsInterceptor {
    return &MetricsInterceptor{
        messagesProcessed: prometheus.NewCounterVec(
            prometheus.CounterOpts{
                Name: "mmate_messages_processed_total",
                Help: "Total number of messages processed",
            },
            []string{"message_type", "status"},
        ),
        processingDuration: prometheus.NewHistogramVec(
            prometheus.HistogramOpts{
                Name: "mmate_processing_duration_seconds",
                Help: "Message processing duration",
                Buckets: prometheus.DefBuckets,
            },
            []string{"message_type"},
        ),
    }
}

func (i *MetricsInterceptor) Intercept(ctx context.Context, 
    msg contracts.Message, next interceptors.Handler) error {
    
    start := time.Now()
    messageType := msg.GetType()
    
    err := next(ctx, msg)
    
    duration := time.Since(start).Seconds()
    status := "success"
    if err != nil {
        status = "error"
    }
    
    i.messagesProcessed.WithLabelValues(messageType, status).Inc()
    i.processingDuration.WithLabelValues(messageType).Observe(duration)
    
    return err
}

// Usage
func main() {
    // Create interceptor pipeline
    pipeline := interceptors.NewPipeline(
        NewMetricsInterceptor(),
        NewLoggingInterceptor(),
        NewTracingInterceptor(),
    )
    
    // Apply to subscriber
    subscriber := messaging.NewMessageSubscriber(
        transport,
        messaging.WithInterceptors(pipeline))
}
```

## Consumer Group Example

```go
package main

import (
    "context"
    "sync"
    "github.com/glimte/mmate-go/messaging"
)

func main() {
    // Create consumer group
    group := messaging.NewConsumerGroup(
        "order-processors",
        messaging.WithGroupSize(5),
        messaging.WithPrefetchCount(2),
    )
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            
            handler := &OrderProcessor{
                workerID: workerID,
                db:       database,
            }
            
            err := group.Subscribe(ctx,
                "cmd.orders.process",
                handler)
            if err != nil {
                log.Printf("Worker %d failed: %v", workerID, err)
            }
        }(i)
    }
    
    // Wait for shutdown
    <-shutdownChan
    group.Stop()
    wg.Wait()
}
```

## Error Handling Example

```go
package main

import (
    "context"
    "errors"
    "github.com/glimte/mmate-go/messaging"
)

// Custom error types
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

type TransientError struct {
    Cause error
}

func (e *TransientError) Error() string {
    return fmt.Sprintf("transient error: %v", e.Cause)
}

func (e *TransientError) Unwrap() error {
    return e.Cause
}

// Handler with error handling
type PaymentHandler struct {
    paymentGateway PaymentGateway
    logger         *slog.Logger
}

func (h *PaymentHandler) Handle(ctx context.Context, msg contracts.Message) error {
    cmd := msg.(*ProcessPaymentCommand)
    
    // Validation errors - don't retry
    if cmd.Amount <= 0 {
        return &ValidationError{
            Field:   "amount",
            Message: "must be positive",
        }
    }
    
    // Process payment
    result, err := h.paymentGateway.Process(ctx, cmd)
    if err != nil {
        // Network errors - retry
        var netErr *NetworkError
        if errors.As(err, &netErr) {
            h.logger.Warn("Network error, will retry",
                "error", err,
                "attempt", msg.GetHeader("retry-count"))
            return &TransientError{Cause: err}
        }
        
        // Rate limit - retry with backoff
        var rateLimitErr *RateLimitError
        if errors.As(err, &rateLimitErr) {
            return messaging.NewTransientError(
                "rate limited", err,
                messaging.WithRetryAfter(rateLimitErr.RetryAfter))
        }
        
        // Unknown error - don't retry
        return messaging.NewPermanentError(
            "payment processing failed", err)
    }
    
    // Publish success event
    event := &PaymentProcessedEvent{
        PaymentID:  result.ID,
        OrderID:    cmd.OrderID,
        Amount:     cmd.Amount,
        ProcessedAt: time.Now(),
    }
    
    return h.publisher.PublishEvent(ctx, event)
}
```

## Health Check Example

```go
package main

import (
    "context"
    "net/http"
    "github.com/glimte/mmate-go/health"
)

type HealthService struct {
    rabbitmq   RabbitMQChecker
    database   DatabaseChecker
    downstream []ServiceChecker
}

func (s *HealthService) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    health := &HealthStatus{
        Status: "healthy",
        Checks: make(map[string]CheckResult),
    }
    
    // Check RabbitMQ
    if err := s.rabbitmq.Check(ctx); err != nil {
        health.Status = "unhealthy"
        health.Checks["rabbitmq"] = CheckResult{
            Status: "unhealthy",
            Error:  err.Error(),
        }
    } else {
        health.Checks["rabbitmq"] = CheckResult{
            Status: "healthy",
        }
    }
    
    // Check database
    if err := s.database.Check(ctx); err != nil {
        health.Status = "degraded"
        health.Checks["database"] = CheckResult{
            Status: "unhealthy",
            Error:  err.Error(),
        }
    }
    
    // Check downstream services
    for _, svc := range s.downstream {
        result := svc.Check(ctx)
        health.Checks[svc.Name()] = result
        if result.Status == "unhealthy" && svc.Critical() {
            health.Status = "unhealthy"
        }
    }
    
    // Set response code
    switch health.Status {
    case "healthy":
        w.WriteHeader(http.StatusOK)
    case "degraded":
        w.WriteHeader(http.StatusOK)
    case "unhealthy":
        w.WriteHeader(http.StatusServiceUnavailable)
    }
    
    json.NewEncoder(w).Encode(health)
}

func main() {
    health := &HealthService{
        rabbitmq: rabbitmqChecker,
        database: dbChecker,
        downstream: []ServiceChecker{
            paymentServiceChecker,
            inventoryServiceChecker,
        },
    }
    
    http.Handle("/health", health)
    http.Handle("/health/live", http.HandlerFunc(
        func(w http.ResponseWriter, r *http.Request) {
            w.WriteHeader(http.StatusOK)
            w.Write([]byte("OK"))
        }))
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Testing Example

```go
package main

import (
    "context"
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/glimte/mmate-go/contracts"
)

// Mock publisher
type MockPublisher struct {
    mock.Mock
}

func (m *MockPublisher) PublishEvent(ctx context.Context, event contracts.Event) error {
    args := m.Called(ctx, event)
    return args.Error(0)
}

// Test handler
func TestOrderHandler_Handle_Success(t *testing.T) {
    // Arrange
    mockPublisher := new(MockPublisher)
    mockOrderService := new(MockOrderService)
    
    handler := &OrderHandler{
        orderService: mockOrderService,
        publisher:    mockPublisher,
    }
    
    cmd := &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand("CreateOrderCommand"),
        CustomerID:  "CUST-123",
        Items: []OrderItem{
            {ProductID: "PROD-1", Quantity: 2, Price: 10.00},
        },
    }
    
    mockOrderService.On("CreateOrder", mock.Anything, cmd).
        Return("ORDER-123", nil)
    
    mockPublisher.On("PublishEvent", mock.Anything, 
        mock.MatchedBy(func(evt contracts.Event) bool {
            e, ok := evt.(*OrderCreatedEvent)
            return ok && e.OrderID == "ORDER-123"
        })).Return(nil)
    
    // Act
    err := handler.Handle(context.Background(), cmd)
    
    // Assert
    assert.NoError(t, err)
    mockOrderService.AssertExpectations(t)
    mockPublisher.AssertExpectations(t)
}

// Integration test
func TestOrderWorkflow_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }
    
    // Setup
    ctx := context.Background()
    
    // Create client with real connections
    client, err := mmate.NewClient("amqp://localhost")
    require.NoError(t, err)
    defer client.Close()
    
    publisher := client.Publisher()
    
    // Test workflow
    workflow := createTestWorkflow()
    context := &OrderFulfillmentContext{
        OrderID:    "TEST-ORDER-123",
        CustomerID: "TEST-CUST-456",
        Items:      testItems,
    }
    
    result, err := workflow.Execute(ctx, context)
    
    assert.NoError(t, err)
    assert.True(t, result.Success)
    assert.NotEmpty(t, context.PaymentID)
    assert.NotEmpty(t, context.ShipmentID)
}
```