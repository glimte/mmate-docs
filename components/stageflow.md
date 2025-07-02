# StageFlow - Workflow Orchestration

StageFlow provides multi-stage workflow orchestration with compensation support, enabling complex business processes across distributed services.

## Overview

StageFlow enables:
- Multi-stage process orchestration
- Automatic compensation on failure
- State persistence between stages
- Distributed execution
- Timeout handling
- Retry policies per stage

## Core Concepts

### Workflow
A sequence of stages that process a context object, with each stage potentially modifying the context.

### Stage
A unit of work that:
- Receives a context
- Performs an operation
- Updates the context
- Can have compensation logic

### Compensation
Rollback logic that runs when a stage fails, undoing previously completed stages in reverse order via dedicated compensation queues.
**Queue-based Architecture**: Both .NET and Go implementations support automatic compensation using queue-based messaging for resilient rollback operations.

### Context
The state object passed between stages, containing both input data and accumulated results.
**Go Implementation**: Uses `map[string]interface{}` for state, with optional typed wrapper available.

## Basic Usage

### Defining a Workflow

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Define context
public class OrderContext : WorkflowContext
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
    
    // Stage results
    public bool InventoryReserved { get; set; }
    public string PaymentId { get; set; }
    public string ShipmentId { get; set; }
}

// Define workflow
public class OrderWorkflow : Workflow<OrderContext>
{
    protected override void Configure(
        IWorkflowBuilder<OrderContext> builder)
    {
        builder
            .AddStage<ValidateOrderStage>()
            .AddStage<CalculatePricingStage>()
            .AddStage<ReserveInventoryStage>()
                .WithCompensation<ReleaseInventoryStage>()
            .AddStage<ProcessPaymentStage>()
                .WithCompensation<RefundPaymentStage>()
            .AddStage<CreateShipmentStage>()
                .WithCompensation<CancelShipmentStage>()
            .AddStage<SendConfirmationStage>();
    }
}
```

</td>
<td>

```go
// OPTION 1: Untyped workflow (current implementation)
func NewOrderWorkflow() *stageflow.Workflow {
    workflow := stageflow.NewWorkflow("order-processing", "Order Processing")
    
    // Add stages with retry policies
    workflow.AddStage("validate", &ValidateOrderStage{},
        stageflow.WithStageTimeout(30*time.Second),
        stageflow.WithStageRetryPolicy(
            reliability.NewExponentialBackoff(100*time.Millisecond, 5*time.Second, 2.0, 3)))
    
    workflow.AddStage("process-payment", &ProcessPaymentStage{},
        stageflow.WithStageTimeout(60*time.Second),
        stageflow.WithStageRetryPolicy(
            reliability.NewLinearBackoff(time.Second, 10*time.Second, 5)))
    
    workflow.AddStage("arrange-shipping", &ArrangeShippingStage{})
    
    return workflow
}

// OPTION 2: Type-safe workflow (via typed wrapper)
type OrderWorkflowContext struct {
    OrderID         string          `json:"orderId"`
    CustomerID      string          `json:"customerId"`
    Items           []OrderItem     `json:"items"`
    TotalAmount     float64         `json:"totalAmount"`
    ShippingAddress ShippingAddress `json:"shippingAddress"`
    
    // Stage results (typed!)
    Validated         bool      `json:"validated,omitempty"`
    InventoryReserved bool      `json:"inventoryReserved,omitempty"`
    PaymentID         string    `json:"paymentId,omitempty"`
    TrackingNumber    string    `json:"trackingNumber,omitempty"`
}

func (c *OrderWorkflowContext) Validate() error {
    if c.CustomerID == "" {
        return fmt.Errorf("customer ID is required")
    }
    return nil
}

func NewTypedOrderWorkflow() *stageflow.Workflow {
    return stageflow.NewTypedWorkflow[*OrderWorkflowContext]("order-processing", "Order Processing").
        AddTypedStage("validate", &ValidateOrderStage{}).
        AddTypedStage("check-inventory", &CheckInventoryStage{}).
        AddTypedStage("process-payment", &ProcessPaymentStage{}).
        AddTypedStage("arrange-shipping", &ArrangeShippingStage{}).
        Build()
}
```

</td>
</tr>
</table>

### Implementing Stages

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Regular stage
public class ProcessPaymentStage : 
    IWorkflowStage<OrderContext>
{
    private readonly IPaymentService _paymentService;
    
    public ProcessPaymentStage(
        IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
    
    public async Task ExecuteAsync(
        OrderContext context,
        CancellationToken cancellationToken)
    {
        var result = await _paymentService
            .ProcessPayment(new PaymentRequest
            {
                Amount = context.TotalAmount,
                CustomerId = context.CustomerId,
                OrderId = context.OrderId
            });
        
        if (!result.Success)
        {
            throw new StageException(
                "Payment failed: " + result.Error);
        }
        
        context.PaymentId = result.TransactionId;
        context.PaymentProcessed = true;
    }
}

// Compensation stage
public class RefundPaymentStage : 
    ICompensationStage<OrderContext>
{
    private readonly IPaymentService _paymentService;
    
    public async Task CompensateAsync(
        OrderContext context,
        CancellationToken cancellationToken)
    {
        if (!string.IsNullOrEmpty(context.PaymentId))
        {
            await _paymentService.RefundPayment(
                context.PaymentId,
                "Order processing failed");
        }
    }
}
```

</td>
<td>

```go
// OPTION 1: Untyped stage (works with map[string]interface{})
type ProcessPaymentStage struct {
    paymentService PaymentService
}

func (s *ProcessPaymentStage) Execute(ctx context.Context, state *stageflow.WorkflowState) (*stageflow.StageResult, error) {
    // Extract data from untyped state
    orderID, _ := state.GlobalData["orderId"].(string)
    totalAmount, _ := state.GlobalData["totalAmount"].(float64)
    customerID, _ := state.GlobalData["customerId"].(string)
    
    result, err := s.paymentService.ProcessPayment(ctx, PaymentRequest{
        Amount:     totalAmount,
        CustomerID: customerID,
        OrderID:    orderID,
    })
    
    if err != nil {
        return &stageflow.StageResult{
            StageID: s.GetStageID(),
            Status:  stageflow.StageFailed,
            Error:   err.Error(),
        }, err
    }
    
    // Update state with results
    state.GlobalData["paymentId"] = result.TransactionID
    state.GlobalData["paymentProcessed"] = true
    
    return &stageflow.StageResult{
        StageID: s.GetStageID(),
        Status:  stageflow.StageCompleted,
    }, nil
}

func (s *ProcessPaymentStage) GetStageID() string {
    return "process-payment"
}

// OPTION 2: Typed stage (via typed wrapper)
type TypedProcessPaymentStage struct{}

func (s *TypedProcessPaymentStage) Execute(ctx context.Context, order *OrderWorkflowContext) error {
    // Direct typed access - no type assertions!
    if order.PaymentID != "" {
        return fmt.Errorf("payment already processed")
    }
    
    // Simulate payment processing
    log.Printf("Processing payment: $%.2f for order %s", order.TotalAmount, order.OrderID)
    time.Sleep(2 * time.Second)
    
    // Update typed context
    order.PaymentID = fmt.Sprintf("PAY-%d", time.Now().Unix())
    
    return nil
}

func (s *TypedProcessPaymentStage) GetStageID() string {
    return "process-payment"
}
```

</td>
</tr>
</table>

### Executing Workflows

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Execute workflow
var workflow = new OrderWorkflow(serviceProvider);
var context = new OrderContext
{
    OrderId = "ORDER-123",
    CustomerId = "CUST-456",
    Items = orderItems
};

try
{
    var result = await workflow.ExecuteAsync(
        context,
        cancellationToken);
    
    if (result.Success)
    {
        logger.LogInformation(
            "Order {OrderId} processed successfully",
            context.OrderId);
    }
    else
    {
        logger.LogError(
            "Order {OrderId} failed: {Error}",
            context.OrderId,
            result.Error);
    }
}
catch (WorkflowException ex)
{
    logger.LogError(ex,
        "Workflow failed at stage {Stage}",
        ex.FailedStage);
}
```

</td>
<td>

```go
// OPTION 1: Untyped workflow execution
workflow := NewOrderWorkflow()
initialData := map[string]interface{}{
    "orderId":    "ORDER-123",
    "customerId": "CUST-456",
    "items":      orderItems,
    "totalAmount": 299.99,
}

// Execute asynchronously via queues
state, err := workflow.Execute(ctx, initialData)
if err != nil {
    log.Printf("Failed to start workflow: %v", err)
    return err
}

log.Printf("Workflow started: %s (processing asynchronously)", state.InstanceID)

// OPTION 2: Typed workflow execution  
workflow := NewTypedOrderWorkflow()
orderContext := &OrderWorkflowContext{
    OrderID:     "ORDER-123",
    CustomerID:  "CUST-456",
    Items:       orderItems,
    TotalAmount: 299.99,
}

// Execute with type safety
state, err := stageflow.ExecuteTyped(workflow, ctx, orderContext)
if err != nil {
    log.Printf("Failed to start typed workflow: %v", err)
    return err
}

log.Printf("Typed workflow started: %s", state.InstanceID)

// OPTION 3: Register and handle via StageFlow engine
engine := stageflow.NewStageFlowEngine(publisher, subscriber)
err = engine.RegisterWorkflow(workflow)
if err != nil {
    return fmt.Errorf("failed to register workflow: %w", err)
}

// Workflows execute via message handling automatically
```

</td>
</tr>
</table>

## Queue-Per-Stage Architecture

StageFlow uses a queue-per-stage architecture that provides natural isolation, concurrency, and resilience:

### How It Works

Each stage in a workflow gets its own dedicated message queue. Messages flow through these queues sequentially, carrying both the payload and the accumulated workflow state.

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Initial   │ --> │ stageflow.      │ --> │ stageflow.      │ --> │ stageflow.      │
│   Message   │     │ workflow.stage0 │     │ workflow.stage1 │     │ workflow.stage2 │
└─────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
                           Queue                   Queue                   Queue
```

### Queue Naming

Queues are named using the pattern: `{StageQueuePrefix}{workflowId}.stage{index}`

**Compensation Queues**: `stageflow.compensation.{workflowId}`
- Used for queue-based compensation messaging
- Ensures reliable rollback operations
- Handles compensation workflows asynchronously

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure stage queue prefix
services.AddStageFlow(options =>
{
    options.StageQueuePrefix = "myapp.stageflow.";
    options.MaxStageConcurrency = 5;
});

// Results in queues like:
// - myapp.stageflow.order-processing.stage0
// - myapp.stageflow.order-processing.stage1
// - myapp.stageflow.order-processing.stage2
// - stageflow.compensation.order-processing (compensation queue)
```

</td>
<td>

```go
// Configure stage queue prefix
config := stageflow.Config{
    StageQueuePrefix:    "myapp.stageflow.",
    MaxStageConcurrency: 5,
}

// Results in queues like:
// - myapp.stageflow.order-processing.stage0
// - myapp.stageflow.order-processing.stage1
// - myapp.stageflow.order-processing.stage2
// - stageflow.compensation.order-processing (compensation queue)
```

</td>
</tr>
</table>

### Message Flow Between Stages

Messages automatically flow from one stage queue to the next:

1. **Stage Execution**: When a stage completes, it publishes a message to the next stage's queue
2. **State Propagation**: The workflow state is serialized and included in the message envelope
3. **Automatic Routing**: The framework handles queue routing - you just define the stages

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// State is automatically propagated
workflow.Stage<OrderRequest>(async (context, state, request) =>
{
    // Process in stage 0
    state.OrderId = Guid.NewGuid().ToString();
    state.Status = "Validated";
    
    // Framework publishes to stage1 queue with state
    await context.Request("next", new InventoryRequest());
});

workflow.Stage<InventoryResponse>(async (context, state, response) =>
{
    // State from stage 0 is available here
    logger.LogInfo($"Processing order {state.OrderId}");
    state.InventoryReserved = response.Success;
    
    // Framework publishes to stage2 queue
});
```

</td>
<td>

```go
// State is automatically propagated
flow.AddStage("validate", func(ctx *StageContext, state *OrderState) error {
    // Process in stage 0
    state.OrderID = uuid.New().String()
    state.Status = "Validated"
    
    // Framework publishes to stage1 queue with state
    return ctx.Request("next", &InventoryRequest{})
})

flow.AddStage("inventory", func(ctx *StageContext, state *OrderState) error {
    // State from stage 0 is available here
    log.Info("Processing order", "orderId", state.OrderID)
    state.InventoryReserved = ctx.Response.(*InventoryResponse).Success
    
    // Framework publishes to stage2 queue
    return nil
})
```

</td>
</tr>
</table>

### Benefits of Queue Isolation

1. **Concurrent Processing**: Multiple instances can process different messages in the same stage
2. **Fault Isolation**: Failure in one stage doesn't affect others
3. **Natural Backpressure**: Each stage can process at its own pace
4. **Message Ordering**: FIFO queues ensure proper message sequence
5. **Debugging**: Easy to inspect messages stuck at specific stages
6. **Scaling**: Scale individual stages based on their workload

### State Persistence in Queues

The workflow state travels with messages through the queues:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class FlowMessageEnvelope<T>
{
    // Message payload
    public T Payload { get; set; }
    
    // Serialized workflow state
    public string SerializedWorkflowState { get; set; }
    public string WorkflowStateType { get; set; }
    
    // Processing checkpoint
    public ProcessingCheckpoint Checkpoint { get; set; }
}

// If pod crashes, message remains in queue
// Next pod picks up message with full state
```

</td>
<td>

```go
type FlowMessageEnvelope struct {
    // Message payload
    Payload json.RawMessage `json:"payload"`
    
    // Serialized workflow state
    SerializedWorkflowState string `json:"serializedWorkflowState"`
    WorkflowStateType      string `json:"workflowStateType"`
    
    // Processing checkpoint
    Checkpoint ProcessingCheckpoint `json:"checkpoint"`
}

// If pod crashes, message remains in queue
// Next pod picks up message with full state
```

</td>
</tr>
</table>

### Queue Configuration

Additional queue-specific configurations:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddStageFlow(options =>
{
    // Queue naming prefix
    options.StageQueuePrefix = "myapp.stages.";
    
    // Max concurrent messages per stage
    options.MaxStageConcurrency = 10;
    
    // Message TTL for stages
    options.StageMessageTTL = TimeSpan.FromHours(1);
    
    // Dead letter after N retries
    options.MaxStageRetries = 3;
});
```

</td>
<td>

```go
config := stageflow.Config{
    // Queue naming prefix
    StageQueuePrefix: "myapp.stages.",
    
    // Max concurrent messages per stage
    MaxStageConcurrency: 10,
    
    // Message TTL for stages
    StageMessageTTL: time.Hour,
    
    // Dead letter after N retries
    MaxStageRetries: 3,
}
```

</td>
</tr>
</table>

### Monitoring Stage Queues

Monitor individual stage queues to identify bottlenecks:

```bash
# List all stage queues
mmate-monitor queue list --prefix stageflow

# Monitor specific stage queue
mmate-monitor queue watch stageflow.order-processing.stage1

# Check queue depths
mmate-monitor queue stats --prefix stageflow
```

This architecture ensures that workflows are resilient to failures, can scale horizontally, and maintain proper message ordering through the entire processing pipeline.

## Advanced Features

### Parallel Stages

Execute multiple stages concurrently:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
builder
    .AddParallelStages(parallel =>
    {
        parallel.AddStage<CheckInventoryStage>();
        parallel.AddStage<ValidatePaymentStage>();
        parallel.AddStage<CheckShippingStage>();
    })
    .AddStage<ProcessOrderStage>();

// Custom parallel stage
public class ParallelValidationStage : 
    IWorkflowStage<OrderContext>
{
    public async Task ExecuteAsync(
        OrderContext context,
        CancellationToken ct)
    {
        var tasks = new[]
        {
            CheckInventoryAsync(context),
            ValidatePaymentAsync(context),
            CheckShippingAsync(context)
        };
        
        await Task.WhenAll(tasks);
    }
}
```

</td>
<td>

```go
// Add parallel stages
flow.AddParallel("validations", 
    []stageflow.Stage[*OrderContext]{
        &CheckInventoryStage{},
        &ValidatePaymentStage{},
        &CheckShippingStage{},
    })

// Custom parallel stage
type ParallelValidationStage struct {
    inventory InventoryService
    payment   PaymentService
    shipping  ShippingService
}

func (s *ParallelValidationStage) Execute(
    ctx context.Context,
    context *OrderContext) error {
    
    g, ctx := errgroup.WithContext(ctx)
    
    g.Go(func() error {
        return s.inventory.Check(ctx, context)
    })
    
    g.Go(func() error {
        return s.payment.Validate(ctx, context)
    })
    
    g.Go(func() error {
        return s.shipping.Check(ctx, context)
    })
    
    return g.Wait()
}
```

</td>
</tr>
</table>

### Conditional Stages

Execute stages based on conditions:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
builder
    .AddStage<CalculatePricingStage>()
    .AddConditionalStage<ApplyDiscountStage>(
        condition: ctx => ctx.TotalAmount > 100)
    .AddConditionalStage<FraudCheckStage>(
        condition: ctx => ctx.TotalAmount > 1000);

// Branch workflow
builder
    .AddBranch(
        condition: ctx => ctx.IsInternational,
        international: branch => branch
            .AddStage<InternationalShippingStage>()
            .AddStage<CustomsStage>(),
        domestic: branch => branch
            .AddStage<DomesticShippingStage>());
```

</td>
<td>

```go
// Conditional stages
flow.AddConditional("apply-discount",
    &ApplyDiscountStage{},
    func(ctx *OrderContext) bool {
        return ctx.TotalAmount > 100
    })

flow.AddConditional("fraud-check",
    &FraudCheckStage{},
    func(ctx *OrderContext) bool {
        return ctx.TotalAmount > 1000
    })

// Branch workflow
flow.AddBranch("shipping-type",
    func(ctx *OrderContext) string {
        if ctx.IsInternational {
            return "international"
        }
        return "domestic"
    },
    map[string][]stageflow.Stage[*OrderContext]{
        "international": {
            &InternationalShippingStage{},
            &CustomsStage{},
        },
        "domestic": {
            &DomesticShippingStage{},
        },
    })
```

</td>
</tr>
</table>

### Stage Retry Policies

Configure retry behavior per stage:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
builder.AddStage<ProcessPaymentStage>()
    .WithRetry(policy => policy
        .MaxAttempts(3)
        .WithExponentialBackoff(
            initialDelay: TimeSpan.FromSeconds(1),
            maxDelay: TimeSpan.FromSeconds(30))
        .RetryOn<TransientException>()
        .AbortOn<ValidationException>());

// Custom retry policy
public class CustomRetryPolicy : IRetryPolicy
{
    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        CancellationToken ct)
    {
        for (int i = 0; i < 3; i++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (
                IsRetryable(ex) && i < 2)
            {
                await Task.Delay(
                    TimeSpan.FromSeconds(
                        Math.Pow(2, i)), ct);
            }
        }
        throw new RetriesExhaustedException();
    }
}
```

</td>
<td>

```go
// Configure retry per stage
flow.AddStage("process-payment",
    &ProcessPaymentStage{},
    stageflow.WithRetry(
        stageflow.MaxAttempts(3),
        stageflow.ExponentialBackoff(
            time.Second, 30*time.Second),
        stageflow.RetryOn(
            func(err error) bool {
                var transientErr *TransientError
                return errors.As(err, &transientErr)
            }),
        stageflow.AbortOn(
            func(err error) bool {
                var validationErr *ValidationError
                return errors.As(err, &validationErr)
            })))

// Custom retry policy
type CustomRetryPolicy struct {
    maxAttempts int
    backoff     func(int) time.Duration
}

func (p *CustomRetryPolicy) Execute(
    ctx context.Context,
    operation func() error) error {
    
    var lastErr error
    for i := 0; i < p.maxAttempts; i++ {
        err := operation()
        if err == nil {
            return nil
        }
        
        if !isRetryable(err) || 
           i == p.maxAttempts-1 {
            return err
        }
        
        lastErr = err
        select {
        case <-time.After(p.backoff(i)):
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    
    return fmt.Errorf(
        "retries exhausted: %w", lastErr)
}
```

</td>
</tr>
</table>

### State Persistence

Persist workflow state between stages:

#### Queue-Based Resilient State Persistence

StageFlow includes built-in queue-based state persistence that enables workflows to survive pod crashes without losing progress:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// State automatically embedded in messages
public class FlowMessageEnvelope<T>
{
    // Complete workflow state
    public string SerializedWorkflowState { get; set; }
    public string WorkflowStateType { get; set; }
    
    // Fine-grained checkpoints
    public ProcessingCheckpoint Checkpoint { get; set; }
}

// Checkpoint within stages
workflow.Stage<OrderCommand>(async (context, state, cmd) => 
{
    // Check if resuming from crash
    if (context.IsStepCompleted("inventory-reserved"))
    {
        var inventory = context.GetStepResult<InventoryResult>("inventory");
        logger.LogInfo($"Resuming with inventory: {inventory.ReservationId}");
    }
    else
    {
        // Reserve inventory
        var result = await ReserveInventory(cmd.Items);
        
        // Save checkpoint
        await context.StoreStepResultAsync("inventory", result);
        await context.CheckpointStepAsync("inventory-reserved");
    }
    
    // Continue with payment...
});
```

</td>
<td>

```go
// Automatic state persistence
flow := stageflow.NewFlow[*OrderContext](
    "order-processing",
    stageflow.WithStateStore(
        stageflow.NewQueueBasedStateStore()))

// State saved after each stage
flow.AddStage("reserve-inventory", 
    func(ctx *StageContext, state *OrderState) error {
        // Check previous results
        if result, ok := ctx.State.StageResults["inventory"]; ok {
            log.Info("Found previous inventory reservation", 
                "id", result.ReservationID)
            return nil
        }
        
        // Process and save result
        result := reserveInventory(state.Items)
        ctx.SaveStageResult("inventory", result)
        
        return nil
    })
```

</td>
</tr>
</table>

#### Traditional State Persistence

For external state stores:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure persistence
services.AddStageFlow(options =>
{
    options.UsePersistence<RedisPersistence>();
    options.EnableCheckpoints = true;
    options.CheckpointInterval = 
        TimeSpan.FromMinutes(1);
});

// Custom persistence
public class RedisPersistence : 
    IWorkflowPersistence
{
    public async Task SaveStateAsync<T>(
        string workflowId,
        string stageId,
        T context)
    {
        var key = $"workflow:{workflowId}:{stageId}";
        var json = JsonSerializer.Serialize(context);
        await _redis.StringSetAsync(
            key, json, TimeSpan.FromHours(24));
    }
    
    public async Task<T> LoadStateAsync<T>(
        string workflowId,
        string stageId)
    {
        var key = $"workflow:{workflowId}:{stageId}";
        var json = await _redis.StringGetAsync(key);
        return json.HasValue 
            ? JsonSerializer.Deserialize<T>(json)
            : default(T);
    }
}
```

</td>
<td>

```go
// Configure persistence
persistence := redis.NewPersistence(redisClient)
flow := stageflow.NewFlow[*OrderContext](
    "order-processing",
    stageflow.WithPersistence(persistence),
    stageflow.WithCheckpoints(true),
    stageflow.WithCheckpointInterval(time.Minute))

// Custom persistence
type RedisPersistence struct {
    client *redis.Client
}

func (p *RedisPersistence) SaveState(
    ctx context.Context,
    workflowID, stageID string,
    state interface{}) error {
    
    key := fmt.Sprintf(
        "workflow:%s:%s", workflowID, stageID)
    
    data, err := json.Marshal(state)
    if err != nil {
        return err
    }
    
    return p.client.Set(ctx, key, data,
        24*time.Hour).Err()
}

func (p *RedisPersistence) LoadState(
    ctx context.Context,
    workflowID, stageID string,
    state interface{}) error {
    
    key := fmt.Sprintf(
        "workflow:%s:%s", workflowID, stageID)
    
    data, err := p.client.Get(ctx, key).Bytes()
    if err == redis.Nil {
        return nil // No state found
    }
    if err != nil {
        return err
    }
    
    return json.Unmarshal(data, state)
}
```

</td>
</tr>
</table>

### Distributed Execution

Execute stages across multiple services:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Distributed workflow
public class DistributedOrderWorkflow : 
    DistributedWorkflow<OrderContext>
{
    protected override void Configure(
        IDistributedWorkflowBuilder<OrderContext> builder)
    {
        builder
            // Local stage
            .AddLocalStage<ValidateOrderStage>()
            
            // Remote stage via messaging
            .AddRemoteStage(
                endpointId: "inventory-service",
                command: ctx => new ReserveInventoryCommand
                {
                    OrderId = ctx.OrderId,
                    Items = ctx.Items
                })
            .OnReply<InventoryReservedReply>(
                (ctx, reply) =>
                {
                    ctx.InventoryReserved = reply.Success;
                    ctx.ReservationId = reply.ReservationId;
                })
            
            // HTTP stage
            .AddHttpStage(
                url: "https://payment.service/process",
                request: ctx => new PaymentRequest
                {
                    Amount = ctx.TotalAmount,
                    OrderId = ctx.OrderId
                })
            .OnResponse<PaymentResponse>(
                (ctx, response) =>
                {
                    ctx.PaymentId = response.TransactionId;
                });
    }
}
```

</td>
<td>

```go
// Distributed workflow
func NewDistributedOrderWorkflow(
    messaging messaging.Publisher) *stageflow.Flow[*OrderContext] {
    
    flow := stageflow.NewDistributedFlow[*OrderContext](
        "order-processing")
    
    // Local stage
    flow.AddStage("validate", 
        &ValidateOrderStage{})
    
    // Remote stage via messaging
    flow.AddRemoteStage("reserve-inventory",
        stageflow.RemoteStageConfig{
            EndpointID: "inventory-service",
            BuildCommand: func(ctx *OrderContext) contracts.Command {
                return &ReserveInventoryCommand{
                    OrderID: ctx.OrderID,
                    Items:   ctx.Items,
                }
            },
            HandleReply: func(ctx *OrderContext, reply contracts.Reply) error {
                invReply := reply.(*InventoryReservedReply)
                ctx.InventoryReserved = invReply.Success
                ctx.ReservationID = invReply.ReservationID
                return nil
            },
        })
    
    // HTTP stage
    flow.AddHTTPStage("process-payment",
        stageflow.HTTPStageConfig{
            URL: "https://payment.service/process",
            BuildRequest: func(ctx *OrderContext) interface{} {
                return PaymentRequest{
                    Amount:  ctx.TotalAmount,
                    OrderID: ctx.OrderID,
                }
            },
            HandleResponse: func(ctx *OrderContext, resp *PaymentResponse) error {
                ctx.PaymentID = resp.TransactionID
                return nil
            },
        })
    
    return flow
}
```

</td>
</tr>
</table>

## Testing Workflows

### Unit Testing Stages

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task ProcessPaymentStage_Success()
{
    // Arrange
    var mockPaymentService = new Mock<IPaymentService>();
    mockPaymentService
        .Setup(x => x.ProcessPayment(It.IsAny<PaymentRequest>()))
        .ReturnsAsync(new PaymentResult
        {
            Success = true,
            TransactionId = "TXN-123"
        });
    
    var stage = new ProcessPaymentStage(
        mockPaymentService.Object);
    
    var context = new OrderContext
    {
        TotalAmount = 100,
        CustomerId = "CUST-123"
    };
    
    // Act
    await stage.ExecuteAsync(context, CancellationToken.None);
    
    // Assert
    Assert.AreEqual("TXN-123", context.PaymentId);
    Assert.IsTrue(context.PaymentProcessed);
}
```

</td>
<td>

```go
func TestProcessPaymentStage_Success(t *testing.T) {
    // Arrange
    mockPaymentService := &MockPaymentService{
        ProcessPaymentFunc: func(
            ctx context.Context,
            req PaymentRequest) (*PaymentResult, error) {
            return &PaymentResult{
                Success:       true,
                TransactionID: "TXN-123",
            }, nil
        },
    }
    
    stage := &ProcessPaymentStage{
        paymentService: mockPaymentService,
    }
    
    context := &OrderContext{
        TotalAmount: 100,
        CustomerID:  "CUST-123",
    }
    
    // Act
    err := stage.Execute(
        context.Background(), context)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, "TXN-123", context.PaymentID)
    assert.True(t, context.PaymentProcessed)
}
```

</td>
</tr>
</table>

### Integration Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task OrderWorkflow_CompleteFlow()
{
    // Arrange
    var services = new ServiceCollection();
    services.AddStageFlow()
        .AddTransient<IPaymentService, MockPaymentService>()
        .AddTransient<IInventoryService, MockInventoryService>();
    
    var provider = services.BuildServiceProvider();
    var workflow = new OrderWorkflow(provider);
    
    var context = new OrderContext
    {
        OrderId = "ORDER-123",
        Items = new[] { 
            new OrderItem { ProductId = "P1", Quantity = 2 } 
        }
    };
    
    // Act
    var result = await workflow.ExecuteAsync(context);
    
    // Assert
    Assert.IsTrue(result.Success);
    Assert.IsTrue(context.InventoryReserved);
    Assert.IsNotNull(context.PaymentId);
    Assert.AreEqual(
        WorkflowStatus.Completed, 
        result.Status);
}
```

</td>
<td>

```go
func TestOrderWorkflow_CompleteFlow(t *testing.T) {
    // Arrange
    workflow := NewOrderWorkflow()
    workflow.RegisterServices(
        &MockPaymentService{},
        &MockInventoryService{})
    
    context := &OrderContext{
        OrderID: "ORDER-123",
        Items: []OrderItem{
            {ProductID: "P1", Quantity: 2},
        },
    }
    
    // Act
    result, err := workflow.Execute(
        context.Background(), context)
    
    // Assert
    assert.NoError(t, err)
    assert.True(t, result.Success)
    assert.True(t, context.InventoryReserved)
    assert.NotEmpty(t, context.PaymentID)
    assert.Equal(t, 
        stageflow.StatusCompleted,
        result.Status)
}
```

</td>
</tr>
</table>

## Monitoring and Observability

### Workflow Metrics

1. **Execution Metrics**
   - Total executions
   - Success/failure rates
   - Average duration
   - Stage durations

2. **Compensation Metrics**
   - Compensation triggers
   - Compensation success rate
   - Rollback duration

3. **Performance Metrics**
   - Queue depths
   - Processing latency
   - Resource utilization

### Logging and Tracing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Structured logging
services.AddStageFlow(options =>
{
    options.UseStructuredLogging();
    options.LogLevel = LogLevel.Information;
});

// Custom logging
public class LoggingStage : IWorkflowStage<OrderContext>
{
    private readonly ILogger<LoggingStage> _logger;
    
    public async Task ExecuteAsync(
        OrderContext context,
        CancellationToken ct)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = context.OrderId,
            ["Stage"] = "ProcessPayment"
        }))
        {
            _logger.LogInformation(
                "Processing payment for order {OrderId}",
                context.OrderId);
            
            // Stage logic...
        }
    }
}
```

</td>
<td>

```go
// Structured logging
logger := slog.New(slog.NewJSONHandler(
    os.Stdout, nil))

flow := stageflow.NewFlow[*OrderContext](
    "order-processing",
    stageflow.WithLogger(logger))

// Custom logging
type LoggingStage struct {
    logger *slog.Logger
}

func (s *LoggingStage) Execute(
    ctx context.Context,
    context *OrderContext) error {
    
    logger := s.logger.With(
        "orderId", context.OrderID,
        "stage", "ProcessPayment")
    
    logger.Info("Processing payment for order")
    
    // Stage logic...
    
    return nil
}
```

</td>
</tr>
</table>

## Best Practices

1. **Stage Design**
   - Keep stages focused and single-purpose
   - Make stages idempotent when possible
   - Handle partial failures gracefully
   - Log important state changes

2. **Compensation**
   - Always implement compensation for non-idempotent operations
   - Test compensation logic thoroughly
   - Make compensations idempotent
   - Log compensation actions

3. **State Management**
   - Keep context objects lightweight
   - Persist critical state between stages
   - Version your context objects
   - Clean up old workflow data

4. **Error Handling**
   - Distinguish between recoverable and non-recoverable errors
   - Use appropriate retry policies
   - Implement circuit breakers for external calls
   - Monitor failed workflows

5. **Performance**
   - Use parallel stages where possible
   - Implement timeouts for all stages
   - Monitor stage execution times
   - Scale based on workflow metrics

## Common Patterns

### Saga Pattern
Long-running transactions with compensation:
- Order processing
- Multi-service transactions
- Distributed workflows

### Pipeline Pattern
Sequential data processing:
- ETL workflows
- Data validation pipelines
- Report generation

### State Machine Pattern
Complex state transitions:
- Approval workflows
- Document processing
- User onboarding

## Next Steps

- Implement [Interceptors](../interceptors/README.md) for cross-cutting concerns
- Add [Monitoring](../monitoring/README.md) to your workflows
- Review [Examples](../../README.md#examples) for complete implementations
- Learn about [Testing Strategies](../../advanced/testing.md)