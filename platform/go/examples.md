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

## StageFlow Workflow with Queue-Based Compensation

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    "github.com/glimte/mmate-go/stageflow"
    "github.com/glimte/mmate-go/contracts"
)

// Workflow context
type OrderFulfillmentContext struct {
    OrderID    string `json:"orderId"`
    CustomerID string `json:"customerId"`
    Items      []OrderItem `json:"items"`
    
    // Stage results
    PaymentID       string `json:"paymentId,omitempty"`
    InventoryReserved bool `json:"inventoryReserved,omitempty"`
    ShipmentID      string `json:"shipmentId,omitempty"`
    TrackingNumber  string `json:"trackingNumber,omitempty"`
}

// Implement TypedWorkflowContext interface
func (c *OrderFulfillmentContext) Validate() error {
    if c.OrderID == "" {
        return fmt.Errorf("order ID is required")
    }
    if c.CustomerID == "" {
        return fmt.Errorf("customer ID is required")
    }
    if len(c.Items) == 0 {
        return fmt.Errorf("order must have at least one item")
    }
    return nil
}

// Stage implementations
type ValidateOrderStage struct{}

func (s *ValidateOrderStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    log.Printf("Validating order: %s", wfCtx.OrderID)
    
    if wfCtx.OrderID == "" {
        return errors.New("order ID is required")
    }
    if len(wfCtx.Items) == 0 {
        return errors.New("order has no items")
    }
    
    log.Printf("Order validation passed: %s", wfCtx.OrderID)
    return nil
}

func (s *ValidateOrderStage) GetStageID() string {
    return "validate-order"
}

type ProcessPaymentStage struct {
    paymentService PaymentService
}

func (s *ProcessPaymentStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    log.Printf("Processing payment for order: %s", wfCtx.OrderID)
    
    paymentID, err := s.paymentService.ProcessPayment(ctx, PaymentRequest{
        OrderID:    wfCtx.OrderID,
        CustomerID: wfCtx.CustomerID,
        Amount:     calculateTotal(wfCtx.Items),
    })
    if err != nil {
        return fmt.Errorf("payment processing failed: %w", err)
    }
    
    wfCtx.PaymentID = paymentID
    log.Printf("Payment processed successfully: %s", paymentID)
    return nil
}

func (s *ProcessPaymentStage) GetStageID() string {
    return "process-payment"
}

// Compensation stage using TypedCompensationHandler interface
type RefundPaymentCompensation struct {
    paymentService PaymentService
}

func (c *RefundPaymentCompensation) Compensate(ctx context.Context, 
    wfCtx *OrderFulfillmentContext, stageErr error) error {
    log.Printf("COMPENSATION: Refunding payment for order %s (error: %v)", 
        wfCtx.OrderID, stageErr)
    
    if wfCtx.PaymentID == "" {
        log.Printf("No payment to refund for order: %s", wfCtx.OrderID)
        return nil
    }
    
    err := c.paymentService.RefundPayment(ctx, wfCtx.PaymentID)
    if err != nil {
        return fmt.Errorf("compensation failed - could not refund payment %s: %w", 
            wfCtx.PaymentID, err)
    }
    
    // Clear payment state
    wfCtx.PaymentID = ""
    log.Printf("Payment compensation completed for order: %s", wfCtx.OrderID)
    return nil
}

func (c *RefundPaymentCompensation) GetStageID() string {
    return "refund-payment"
}

type ReserveInventoryStage struct {
    inventoryService InventoryService
}

func (s *ReserveInventoryStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    log.Printf("Reserving inventory for order: %s", wfCtx.OrderID)
    
    err := s.inventoryService.ReserveItems(ctx, wfCtx.OrderID, wfCtx.Items)
    if err != nil {
        return fmt.Errorf("inventory reservation failed: %w", err)
    }
    
    wfCtx.InventoryReserved = true
    log.Printf("Inventory reserved successfully for order: %s", wfCtx.OrderID)
    return nil
}

func (s *ReserveInventoryStage) GetStageID() string {
    return "reserve-inventory"
}

// Compensation for inventory reservation
type ReleaseInventoryCompensation struct {
    inventoryService InventoryService
}

func (c *ReleaseInventoryCompensation) Compensate(ctx context.Context, 
    wfCtx *OrderFulfillmentContext, stageErr error) error {
    log.Printf("COMPENSATION: Releasing inventory for order %s (error: %v)", 
        wfCtx.OrderID, stageErr)
    
    if !wfCtx.InventoryReserved {
        log.Printf("No inventory to release for order: %s", wfCtx.OrderID)
        return nil
    }
    
    err := c.inventoryService.ReleaseItems(ctx, wfCtx.OrderID)
    if err != nil {
        return fmt.Errorf("compensation failed - could not release inventory for order %s: %w", 
            wfCtx.OrderID, err)
    }
    
    // Clear inventory state
    wfCtx.InventoryReserved = false
    log.Printf("Inventory compensation completed for order: %s", wfCtx.OrderID)
    return nil
}

func (c *ReleaseInventoryCompensation) GetStageID() string {
    return "release-inventory"
}

type CreateShipmentStage struct {
    shippingService ShippingService
}

func (s *CreateShipmentStage) Execute(ctx context.Context, wfCtx *OrderFulfillmentContext) error {
    log.Printf("Creating shipment for order: %s", wfCtx.OrderID)
    
    shipmentID, trackingNumber, err := s.shippingService.CreateShipment(ctx, ShipmentRequest{
        OrderID:    wfCtx.OrderID,
        CustomerID: wfCtx.CustomerID,
        Items:      wfCtx.Items,
    })
    if err != nil {
        return fmt.Errorf("shipment creation failed: %w", err)
    }
    
    wfCtx.ShipmentID = shipmentID
    wfCtx.TrackingNumber = trackingNumber
    log.Printf("Shipment created successfully: %s (tracking: %s)", shipmentID, trackingNumber)
    return nil
}

func (s *CreateShipmentStage) GetStageID() string {
    return "create-shipment"
}

// Workflow setup with queue-based compensation
func SetupOrderWorkflow(engine *stageflow.StageFlowEngine) error {
    // Create typed workflow with compensation support
    workflow := stageflow.NewTypedWorkflow[*OrderFulfillmentContext](
        "order-fulfillment", 
        "Order Fulfillment Workflow").
        AddTypedStage("validate", &ValidateOrderStage{}).
        AddTypedStage("payment", &ProcessPaymentStage{
            paymentService: paymentService,
        }).
            WithCompensation(&RefundPaymentCompensation{
                paymentService: paymentService,
            }).
        AddTypedStage("inventory", &ReserveInventoryStage{
            inventoryService: inventoryService,
        }).
            WithCompensation(&ReleaseInventoryCompensation{
                inventoryService: inventoryService,
            }).
        AddTypedStage("shipping", &CreateShipmentStage{
            shippingService: shippingService,
        }).
        Build()
    
    // Register workflow with engine (enables queue-based execution)
    return engine.RegisterWorkflow(workflow)
}

// Execute workflow via StageFlow engine (queue-based)
func ProcessOrderAsync(ctx context.Context, engine *stageflow.StageFlowEngine, 
    orderID string, customerID string, items []OrderItem) error {
    
    // Create workflow context
    orderContext := &OrderFulfillmentContext{
        OrderID:    orderID,
        CustomerID: customerID,
        Items:      items,
    }
    
    // Start workflow execution (returns immediately, processes asynchronously via queues)
    state, err := stageflow.ExecuteTyped(workflow, ctx, orderContext)
    if err != nil {
        return fmt.Errorf("failed to start workflow: %w", err)
    }
    
    log.Printf("Order workflow started: %s (instance: %s)", orderID, state.InstanceID)
    
    // Workflow continues processing asynchronously via stage queues
    // Compensation will be handled automatically via compensation queue if needed
    return nil
}

// Direct execution (synchronous) - for testing or simple cases
func ProcessOrderSync(ctx context.Context, orderID string, customerID string, items []OrderItem) error {
    // Create workflow
    workflow := stageflow.NewTypedWorkflow[*OrderFulfillmentContext](
        "order-fulfillment-sync", 
        "Synchronous Order Fulfillment").
        AddTypedStage("validate", &ValidateOrderStage{}).
        AddTypedStage("payment", &ProcessPaymentStage{
            paymentService: paymentService,
        }).
            WithCompensation(&RefundPaymentCompensation{
                paymentService: paymentService,
            }).
        AddTypedStage("inventory", &ReserveInventoryStage{
            inventoryService: inventoryService,
        }).
            WithCompensation(&ReleaseInventoryCompensation{
                inventoryService: inventoryService,
            }).
        AddTypedStage("shipping", &CreateShipmentStage{
            shippingService: shippingService,
        }).
        Build()
    
    // Create context
    orderContext := &OrderFulfillmentContext{
        OrderID:    orderID,
        CustomerID: customerID,
        Items:      items,
    }
    
    // Execute synchronously
    result, err := workflow.Execute(ctx, orderContext)
    if err != nil {
        log.Printf("Workflow failed: %v", err)
        if result != nil && len(result.Compensations) > 0 {
            log.Printf("Compensations executed: %v", result.Compensations)
        }
        return err
    }
    
    log.Printf("Order processed successfully: %+v", result)
    return nil
}
```

**Key Features of Go Compensation Implementation:**

1. **Queue-Based Architecture**: Compensation runs asynchronously via dedicated `stageflow.compensation.{workflowId}` queues
2. **TypedCompensationHandler Interface**: Type-safe compensation handlers with proper error handling  
3. **Automatic Rollback**: Failed workflows trigger automatic compensation in reverse stage order
4. **Resilient Processing**: Compensation messages persist in queues, surviving pod crashes
5. **Event Publishing**: `WorkflowCompensatedEvent` published when compensation completes
6. **Both Sync and Async**: Supports both direct execution and queue-based execution via StageFlowEngine

## Interceptor Example

```go
package main

import (
    "context"
    "time"
    "github.com/glimte/mmate-go/interceptors"
    "github.com/glimte/mmate-go/contracts"
)

// Using the built-in metrics collector from monitor package
import (
    "github.com/glimte/mmate-go/monitor"
    "github.com/glimte/mmate-go/interceptors"
)

func setupMetrics() {
    // Use the monitor package's SimpleMetricsCollector
    collector := monitor.NewSimpleMetricsCollector()
    metricsInterceptor := interceptors.NewMetricsInterceptor(collector)
    
    // Use in pipeline
    pipeline := interceptors.NewPipeline(
        metricsInterceptor,
        // other interceptors...
    )
    
    // Later, get metrics summary
    summary := collector.GetMetricsSummary()
    fmt.Printf("Message counts: %v\n", summary.MessageCounts)
    fmt.Printf("Processing stats: %v\n", summary.ProcessingStats)
}

// Usage
func main() {
    // Setup metrics collection
    setupMetrics()
    
    // Or use the built-in simple metrics collector
    client, err := mmate.NewClientWithOptions(connectionString,
        mmate.WithDefaultMetrics())
    
    // Get metrics summary
    summary := client.GetMetricsSummary()
    if summary != nil {
        fmt.Printf("Messages processed: %v\n", summary.MessageCounts)
    }
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

## Contract Publishing Example

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

// Service command with full documentation
type CreateOrderCommand struct {
    contracts.BaseCommand
    CustomerID   string  `json:"customerId" description:"Unique customer identifier"`
    ProductID    string  `json:"productId" description:"Product to order"`
    Quantity     int     `json:"quantity" description:"Number of items"`
    TotalAmount  float64 `json:"totalAmount" description:"Order total in USD"`
}

// Service reply
type OrderCreatedReply struct {
    contracts.BaseReply
    OrderID     string    `json:"orderId"`
    Status      string    `json:"status"`
    CreatedAt   time.Time `json:"createdAt"`
}

func main() {
    ctx := context.Background()
    
    // Create client with contract publishing enabled
    client, err := mmate.NewClientWithOptions(
        "amqp://localhost",
        mmate.WithServiceName("order-service"),
        mmate.WithContractPublishing(), // Enable auto-publishing
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
    
    // Register message types
    messaging.Register("CreateOrderCommand", func() contracts.Message { 
        return &CreateOrderCommand{} 
    })
    messaging.Register("OrderCreatedReply", func() contracts.Message { 
        return &OrderCreatedReply{} 
    })
    
    dispatcher := client.Dispatcher()
    
    // Register handler - contract is automatically extracted and published
    handler := messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
        cmd := msg.(*CreateOrderCommand)
        log.Printf("Processing order for customer %s", cmd.CustomerID)
        
        // Process order...
        
        // Send reply
        reply := &OrderCreatedReply{
            BaseReply: contracts.BaseReply{
                BaseMessage: contracts.NewBaseMessage("OrderCreatedReply"),
                Success:     true,
            },
            OrderID:   "ORDER-12345",
            Status:    "created",
            CreatedAt: time.Now(),
        }
        
        if cmd.ReplyTo != "" {
            return client.Publisher().PublishReply(ctx, reply, cmd.ReplyTo)
        }
        return nil
    })
    
    // Register handler - this triggers contract extraction and publishing
    err = dispatcher.RegisterHandler(&CreateOrderCommand{}, handler)
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("Service started with contract publishing enabled")
    log.Println("Contracts are published to 'mmate.contracts' exchange")
    
    // The published contract includes:
    // - Endpoint ID: "order-service.CreateOrderCommand"
    // - Input/Output types with JSON schemas
    // - Queue information
    // - Service metadata
    
    // Keep service running
    select {}
}
```

### Consuming Published Contracts

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    mmate "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/contracts"
    "github.com/glimte/mmate-go/messaging"
)

func main() {
    ctx := context.Background()
    
    // Create client to consume contracts
    client, err := mmate.NewClientWithOptions(
        "amqp://localhost",
        mmate.WithServiceName("api-gateway"),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
    
    // Register contract message type
    messaging.Register("ContractAnnouncement", func() contracts.Message { 
        return &contracts.ContractAnnouncement{} 
    })
    
    // Create queue for receiving contract announcements
    viewerQueue := "contract.viewer.api-gateway"
    transport := client.Transport()
    
    err = transport.DeclareQueueWithBindings(ctx, viewerQueue, 
        messaging.QueueOptions{
            Durable:    true,
            AutoDelete: false,
        }, 
        []messaging.QueueBinding{
            {
                Exchange:   "mmate.contracts",
                RoutingKey: "contract.announce",
            },
        })
    if err != nil {
        log.Fatal(err)
    }
    
    // Subscribe to contract announcements
    subscriber := client.Subscriber()
    handler := messaging.MessageHandlerFunc(func(ctx context.Context, msg contracts.Message) error {
        announcement := msg.(*contracts.ContractAnnouncement)
        
        for _, contract := range announcement.Contracts {
            log.Printf("Discovered service contract:")
            log.Printf("  Service: %s", contract.ServiceName)
            log.Printf("  Endpoint: %s", contract.EndpointID)
            log.Printf("  Queue: %s", contract.Queue)
            log.Printf("  Input: %s", contract.InputType)
            log.Printf("  Output: %s", contract.OutputType)
            
            // Use the schema for validation
            if len(contract.InputSchema) > 0 {
                var schema map[string]interface{}
                json.Unmarshal(contract.InputSchema, &schema)
                log.Printf("  Schema available for validation")
            }
        }
        
        return nil
    })
    
    err = subscriber.Subscribe(ctx, viewerQueue, "ContractAnnouncement",
        messaging.NewAutoAcknowledgingHandler(handler))
    if err != nil {
        log.Fatal(err)
    }
    
    log.Println("Listening for service contracts...")
    select {}
}
```