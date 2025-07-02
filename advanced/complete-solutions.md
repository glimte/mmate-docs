# Complete Solutions - Production Examples

This guide demonstrates complete, production-ready solutions using multiple Mmate components working together to solve real business problems.

## E-commerce Platform

### Overview
A full e-commerce order processing system demonstrating all Mmate components working together.

### Architecture

```
┌─────────────┐     ┌───────────────┐     ┌────────────────┐
│   REST API  │────▶│SyncAsyncBridge│────▶│ Message Broker │
└─────────────┘     └───────────────┘     └───────┬────────┘
                                                  │
                    ┌─────────────────────────────┼──────────────────────────────┐
                    │                             │                              │
              ┌─────▼──────┐             ┌────────▼────────┐           ┌─────────▼────────┐
              │   Orders   │             │    Inventory    │           │     Payment      │
              │   Service  │             │    Service      │           │     Service      │
              └─────┬──────┘             └────────┬────────┘           └─────────┬────────┘
                    │                             │                              │
                    └──────────────────┬──────────┴──────────────────────────────┘
                                       │
                                ┌──────▼──────┐
                                │  StageFlow  │
                                │  Workflow   │
                                └─────────────┘
```

### Implementation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configuration
public class EcommerceConfiguration
{
    public static void ConfigureServices(
        IServiceCollection services, 
        IConfiguration configuration)
    {
        // Core messaging
        services.AddMmate(options =>
        {
            options.ConnectionString = 
                configuration.GetConnectionString("RabbitMQ");
            options.PrefetchCount = 10;
        });
        
        // StageFlow workflows
        services.AddStageFlow(options =>
        {
            options.StageQueuePrefix = "ecommerce.workflow.";
            options.MaxStageConcurrency = 10;
        });
        
        // Interceptors
        services.AddMmateInterceptor<TracingInterceptor>();
        services.AddMmateInterceptor<ValidationInterceptor>();
        services.AddMmateInterceptor<MetricsInterceptor>();
        
        // SyncAsyncBridge for APIs
        services.AddSyncAsyncBridge(options =>
        {
            options.DefaultTimeout = TimeSpan.FromSeconds(30);
            options.MaxConcurrentRequests = 200;
        });
        
        // Schema validation
        services.AddMmateSchema(options =>
        {
            options.ValidationMode = SchemaValidationMode.Strict;
        });
        
        // Application services
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IInventoryService, InventoryService>();
        services.AddScoped<IPaymentService, PaymentService>();
        services.AddSingleton<OrderWorkflow>();
    }
}

// Order Processing Workflow
public class OrderWorkflow : Workflow<OrderContext>
{
    public override string Id => "order-processing";
    
    protected override void Configure(
        IWorkflowBuilder<OrderContext> builder)
    {
        builder
            .AddStage<ValidateOrderStage>()
            
            .AddStage<CheckInventoryStage>()
                .WithCompensation<ReleaseInventoryStage>()
                .WithTimeout(TimeSpan.FromSeconds(20))
            
            .AddStage<ProcessPaymentStage>()
                .WithCompensation<RefundPaymentStage>()
                .WithRetry(policy => policy
                    .MaxAttempts(3)
                    .WithExponentialBackoff())
            
            .AddStage<CreateShipmentStage>()
                .WithTimeout(TimeSpan.FromMinutes(2))
            
            .AddStage<NotifyCustomerStage>()
                .ContinueOnFailure();
    }
}

// Workflow Context
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

// Stage Implementation
public class ProcessPaymentStage : IWorkflowStage<OrderContext>
{
    private readonly IPaymentService _paymentService;
    private readonly ILogger<ProcessPaymentStage> _logger;
    
    public ProcessPaymentStage(
        IPaymentService paymentService,
        ILogger<ProcessPaymentStage> logger)
    {
        _paymentService = paymentService;
        _logger = logger;
    }
    
    public async Task ExecuteAsync(
        OrderContext context, 
        CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "Processing payment for order {OrderId}", 
            context.OrderId);
        
        var result = await _paymentService.ProcessPaymentAsync(
            new PaymentRequest
            {
                OrderId = context.OrderId,
                CustomerId = context.CustomerId,
                Amount = context.TotalAmount
            },
            cancellationToken);
        
        if (!result.Success)
        {
            throw new PaymentException(result.Error);
        }
        
        context.PaymentId = result.PaymentId;
    }
}

// REST API Controller
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly ISyncAsyncBridge _bridge;
    private readonly IMessagePublisher _publisher;
    
    public OrdersController(
        ISyncAsyncBridge bridge,
        IMessagePublisher publisher)
    {
        _bridge = bridge;
        _publisher = publisher;
    }
    
    [HttpPost]
    public async Task<ActionResult<OrderResponse>> CreateOrder(
        CreateOrderRequest request)
    {
        // Validate request
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }
        
        // Use bridge for synchronous response
        var response = await _bridge.SendAndWaitAsync<OrderResponse>(
            new CreateOrderCommand
            {
                CustomerId = request.CustomerId,
                Items = request.Items.Select(i => new OrderItem
                {
                    ProductId = i.ProductId,
                    Quantity = i.Quantity,
                    UnitPrice = i.UnitPrice
                }).ToList()
            },
            TimeSpan.FromSeconds(60));
        
        if (response.Success)
        {
            // Publish event for async processing
            await _publisher.PublishEventAsync(
                new OrderCreatedEvent
                {
                    OrderId = response.OrderId,
                    CustomerId = request.CustomerId,
                    TotalAmount = response.TotalAmount,
                    CreatedAt = DateTime.UtcNow
                });
            
            return Ok(response);
        }
        
        return BadRequest(response.Error);
    }
}
```

</td>
<td>

```go
// Configuration
func ConfigureEcommerce(cfg *config.Config) (*EcommerceSystem, error) {
    // Core messaging
    connManager := rabbitmq.NewConnectionManager(cfg.RabbitMQURL)
    channelPool, err := rabbitmq.NewChannelPool(connManager)
    if err != nil {
        return nil, err
    }
    
    // Create publishers and subscribers
    publisher := messaging.NewMessagePublisher(
        rabbitmq.NewPublisher(channelPool))
    subscriber := messaging.NewMessageSubscriber(
        rabbitmq.NewSubscriber(channelPool))
    
    // StageFlow engine
    stageflowEngine := stageflow.NewEngine(
        publisher, subscriber,
        stageflow.WithQueuePrefix("ecommerce.workflow."),
        stageflow.WithMaxConcurrency(10))
    
    // Interceptors
    pipeline := interceptors.NewPipeline(
        NewTracingInterceptor(),
        NewValidationInterceptor(),
        NewMetricsInterceptor(),
    )
    
    // Bridge for APIs
    bridge, err := bridge.NewBridge(
        publisher, subscriber, logger,
        bridge.WithTimeout(30*time.Second),
        bridge.WithMaxConcurrent(200))
    if err != nil {
        return nil, err
    }
    
    // Schema registry
    schemaRegistry := schema.NewRegistry()
    RegisterSchemas(schemaRegistry)
    
    // Application services
    orderService := NewOrderService(db, publisher)
    inventoryService := NewInventoryService(db)
    paymentService := NewPaymentService(paymentGateway)
    
    // Create workflow
    orderWorkflow := NewOrderWorkflow(
        orderService, 
        inventoryService, 
        paymentService)
    
    return &EcommerceSystem{
        Publisher:    publisher,
        Subscriber:   subscriber,
        Bridge:       bridge,
        Workflow:     orderWorkflow,
        Pipeline:     pipeline,
    }, nil
}

// Order Processing Workflow
type OrderWorkflow struct {
    orderService     *OrderService
    inventoryService *InventoryService
    paymentService   *PaymentService
}

func (w *OrderWorkflow) Configure() *stageflow.Flow[*OrderContext] {
    flow := stageflow.NewFlow[*OrderContext](
        "order-processing",
        stageflow.WithTimeout(10*time.Minute))
    
    flow.AddStage("validate", &ValidateOrderStage{})
    
    flow.AddStage("inventory", &CheckInventoryStage{
        service: w.inventoryService,
    })
    flow.AddCompensation("inventory", &ReleaseInventoryStage{
        service: w.inventoryService,
    })
    
    flow.AddStage("payment", &ProcessPaymentStage{
        service: w.paymentService,
    })
    flow.AddCompensation("payment", &RefundPaymentStage{
        service: w.paymentService,
    })
    
    flow.AddStage("shipment", &CreateShipmentStage{})
    flow.AddStage("notify", &NotifyCustomerStage{})
    
    return flow
}

// Workflow Context
type OrderContext struct {
    OrderID    string
    CustomerID string
    Items      []OrderItem
    TotalAmount float64
    
    // Stage results
    InventoryReserved bool
    PaymentID        string
    ShipmentID       string
}

// Stage Implementation
type ProcessPaymentStage struct {
    service *PaymentService
}

func (s *ProcessPaymentStage) Execute(
    ctx context.Context, 
    wfCtx *OrderContext) error {
    
    log.Printf("Processing payment for order %s", wfCtx.OrderID)
    
    result, err := s.service.ProcessPayment(ctx, PaymentRequest{
        OrderID:    wfCtx.OrderID,
        CustomerID: wfCtx.CustomerID,
        Amount:     wfCtx.TotalAmount,
    })
    
    if err != nil {
        return fmt.Errorf("payment failed: %w", err)
    }
    
    if !result.Success {
        return &PaymentError{Reason: result.Error}
    }
    
    wfCtx.PaymentID = result.PaymentID
    return nil
}

// REST API Handler
type OrderHandler struct {
    bridge    *bridge.Bridge
    publisher messaging.Publisher
}

func (h *OrderHandler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var request CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Validate request
    if err := request.Validate(); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Use bridge for synchronous response
    cmd := &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand("CreateOrderCommand"),
        CustomerID:  request.CustomerID,
        Items:       make([]OrderItem, len(request.Items)),
    }
    
    for i, item := range request.Items {
        cmd.Items[i] = OrderItem{
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
            UnitPrice: item.UnitPrice,
        }
    }
    
    reply, err := h.bridge.SendAndWait(
        r.Context(), cmd, "cmd.orders.create", 60*time.Second)
    
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    response := reply.(*OrderResponse)
    if response.Success {
        // Publish event for async processing
        event := &OrderCreatedEvent{
            BaseEvent:   contracts.NewBaseEvent("OrderCreatedEvent", response.OrderID),
            OrderID:     response.OrderID,
            CustomerID:  request.CustomerID,
            TotalAmount: response.TotalAmount,
            CreatedAt:   time.Now(),
        }
        
        if err := h.publisher.PublishEvent(r.Context(), event); err != nil {
            log.Printf("Failed to publish event: %v", err)
        }
        
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(response)
    } else {
        http.Error(w, response.Error, http.StatusBadRequest)
    }
}
```

</td>
</tr>
</table>

## Financial Services Platform

### Transaction Processing System

A high-reliability financial transaction processing system with audit trails and compliance.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Transaction workflow with compliance checks
public class TransactionWorkflow : Workflow<TransactionContext>
{
    protected override void Configure(
        IWorkflowBuilder<TransactionContext> builder)
    {
        builder
            // Compliance and fraud checks
            .AddStage<ComplianceCheckStage>()
                .WithTimeout(TimeSpan.FromSeconds(10))
            
            .AddStage<FraudDetectionStage>()
                .WithRetry(policy => policy
                    .MaxAttempts(2)
                    .WithConstantDelay(TimeSpan.FromSeconds(1)))
            
            // Account validation
            .AddStage<ValidateAccountsStage>()
            
            // Transaction execution with compensation
            .AddStage<DebitSourceAccountStage>()
                .WithCompensation<CreditSourceAccountStage>()
            
            .AddStage<CreditDestinationAccountStage>()
                .WithCompensation<DebitDestinationAccountStage>()
            
            // Audit and notification
            .AddStage<RecordAuditTrailStage>()
                .ContinueOnFailure()
            
            .AddStage<SendNotificationsStage>()
                .ContinueOnFailure();
    }
}

// Compliance interceptor for all financial messages
public class ComplianceInterceptor : IMessageInterceptor
{
    private readonly IComplianceService _compliance;
    private readonly IAuditLogger _auditLogger;
    
    public async Task OnBeforeConsumeAsync(
        MessageContext context, 
        CancellationToken cancellationToken)
    {
        if (context.Message is IFinancialMessage financial)
        {
            // Log all financial messages
            await _auditLogger.LogMessageAsync(
                context.MessageId,
                financial.GetType().Name,
                financial.Amount,
                financial.Currency);
            
            // Check compliance rules
            var result = await _compliance.CheckAsync(financial);
            if (!result.IsCompliant)
            {
                throw new ComplianceException(result.Reason);
            }
        }
    }
}
```

</td>
<td>

```go
// Transaction workflow with compliance checks
func NewTransactionWorkflow(
    compliance ComplianceService,
    accounts AccountService) *stageflow.Flow[*TransactionContext] {
    
    flow := stageflow.NewFlow[*TransactionContext](
        "transaction-processing")
    
    // Compliance and fraud checks
    flow.AddStage("compliance", &ComplianceCheckStage{
        service: compliance,
    })
    
    flow.AddStage("fraud", &FraudDetectionStage{})
    
    // Account validation
    flow.AddStage("validate", &ValidateAccountsStage{
        accounts: accounts,
    })
    
    // Transaction execution with compensation
    flow.AddStage("debit", &DebitSourceAccountStage{
        accounts: accounts,
    })
    flow.AddCompensation("debit", &CreditSourceAccountStage{
        accounts: accounts,
    })
    
    flow.AddStage("credit", &CreditDestinationAccountStage{
        accounts: accounts,
    })
    flow.AddCompensation("credit", &DebitDestinationAccountStage{
        accounts: accounts,
    })
    
    // Audit and notification
    flow.AddStage("audit", &RecordAuditTrailStage{})
    flow.AddStage("notify", &SendNotificationsStage{})
    
    return flow
}

// Compliance interceptor for all financial messages
type ComplianceInterceptor struct {
    compliance  ComplianceService
    auditLogger AuditLogger
}

func (i *ComplianceInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    if financial, ok := msg.(FinancialMessage); ok {
        // Log all financial messages
        err := i.auditLogger.LogMessage(ctx, AuditEntry{
            MessageID: msg.GetID(),
            Type:      msg.GetType(),
            Amount:    financial.GetAmount(),
            Currency:  financial.GetCurrency(),
        })
        if err != nil {
            return err
        }
        
        // Check compliance rules
        result, err := i.compliance.Check(ctx, financial)
        if err != nil {
            return err
        }
        if !result.IsCompliant {
            return &ComplianceError{Reason: result.Reason}
        }
    }
    
    return next(ctx, msg)
}
```

</td>
</tr>
</table>

## Healthcare Management System

### Patient Record Processing

A HIPAA-compliant patient record management system with encryption and access control.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// HIPAA compliance through interceptors
public class HIPAAInterceptor : IMessageInterceptor
{
    private readonly IEncryptionService _encryption;
    private readonly IAccessControlService _accessControl;
    private readonly IAuditService _audit;
    
    public async Task OnBeforePublishAsync(
        MessageContext context, 
        CancellationToken cancellationToken)
    {
        if (context.Message is IPatientData patientData)
        {
            // Encrypt PII/PHI data
            patientData.EncryptedData = await _encryption
                .EncryptAsync(patientData.GetSensitiveData());
            patientData.ClearSensitiveData();
            
            // Add access control metadata
            context.Headers["hipaa-classification"] = "PHI";
            context.Headers["encryption-key-id"] = 
                _encryption.CurrentKeyId;
        }
    }
    
    public async Task OnBeforeConsumeAsync(
        MessageContext context, 
        CancellationToken cancellationToken)
    {
        if (context.Message is IPatientData patientData)
        {
            // Check access permissions
            var principal = context.GetPrincipal();
            if (!await _accessControl.CanAccessPHI(principal))
            {
                throw new UnauthorizedAccessException(
                    "Access to PHI denied");
            }
            
            // Decrypt data
            var keyId = context.Headers["encryption-key-id"];
            patientData.SetSensitiveData(
                await _encryption.DecryptAsync(
                    patientData.EncryptedData, keyId));
            
            // Audit access
            await _audit.LogAccessAsync(
                principal.Identity.Name,
                context.MessageId,
                "PHI_ACCESS");
        }
    }
}

// Patient record workflow
public class PatientAdmissionWorkflow : 
    Workflow<PatientAdmissionContext>
{
    protected override void Configure(
        IWorkflowBuilder<PatientAdmissionContext> builder)
    {
        builder
            .AddStage<VerifyInsuranceStage>()
                .WithTimeout(TimeSpan.FromMinutes(2))
            
            .AddStage<AssignRoomStage>()
                .WithRetry(policy => policy.MaxAttempts(3))
            
            .AddStage<CreateMedicalRecordStage>()
            
            .AddStage<NotifyMedicalStaffStage>()
            
            .AddStage<ScheduleInitialAssessmentStage>();
    }
}
```

</td>
<td>

```go
// HIPAA compliance through interceptors
type HIPAAInterceptor struct {
    encryption   EncryptionService
    accessControl AccessControlService
    audit        AuditService
}

func (i *HIPAAInterceptor) InterceptPublish(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    if patientData, ok := msg.(PatientData); ok {
        // Encrypt PII/PHI data
        encrypted, keyID, err := i.encryption.Encrypt(
            patientData.GetSensitiveData())
        if err != nil {
            return err
        }
        
        patientData.SetEncryptedData(encrypted)
        patientData.ClearSensitiveData()
        
        // Add access control metadata
        msg.SetHeader("hipaa-classification", "PHI")
        msg.SetHeader("encryption-key-id", keyID)
    }
    
    return next(ctx, msg)
}

func (i *HIPAAInterceptor) InterceptConsume(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    if patientData, ok := msg.(PatientData); ok {
        // Check access permissions
        principal := GetPrincipal(ctx)
        canAccess, err := i.accessControl.CanAccessPHI(
            ctx, principal)
        if err != nil {
            return err
        }
        if !canAccess {
            return errors.New("access to PHI denied")
        }
        
        // Decrypt data
        keyID := msg.GetHeader("encryption-key-id")
        decrypted, err := i.encryption.Decrypt(
            patientData.GetEncryptedData(), keyID)
        if err != nil {
            return err
        }
        patientData.SetSensitiveData(decrypted)
        
        // Audit access
        err = i.audit.LogAccess(ctx, AuditEntry{
            User:      principal.Name,
            MessageID: msg.GetID(),
            Action:    "PHI_ACCESS",
        })
        if err != nil {
            return err
        }
    }
    
    return next(ctx, msg)
}

// Patient record workflow
func NewPatientAdmissionWorkflow() *stageflow.Flow[*PatientAdmissionContext] {
    flow := stageflow.NewFlow[*PatientAdmissionContext](
        "patient-admission")
    
    flow.AddStage("insurance", &VerifyInsuranceStage{})
    flow.AddStage("room", &AssignRoomStage{})
    flow.AddStage("record", &CreateMedicalRecordStage{})
    flow.AddStage("notify", &NotifyMedicalStaffStage{})
    flow.AddStage("assessment", &ScheduleInitialAssessmentStage{})
    
    return flow
}
```

</td>
</tr>
</table>

## IoT Data Processing Platform

### High-Volume Sensor Data Processing

A platform for processing millions of IoT sensor readings with real-time analytics.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Batch processing for high throughput
public class SensorDataBatchHandler : 
    IMessageHandler<SensorDataBatch>
{
    private readonly ITimeSeriesDatabase _tsdb;
    private readonly IAnalyticsEngine _analytics;
    
    public async Task HandleAsync(
        SensorDataBatch batch, 
        MessageContext context)
    {
        // Validate batch
        var validReadings = batch.Readings
            .Where(r => r.IsValid())
            .ToList();
        
        // Store in time-series database
        await _tsdb.WriteBatchAsync(validReadings);
        
        // Real-time analytics
        var anomalies = await _analytics
            .DetectAnomaliesAsync(validReadings);
        
        // Publish alerts for anomalies
        foreach (var anomaly in anomalies)
        {
            await _publisher.PublishEventAsync(
                new AnomalyDetectedEvent
                {
                    DeviceId = anomaly.DeviceId,
                    ReadingType = anomaly.Type,
                    Value = anomaly.Value,
                    Threshold = anomaly.Threshold,
                    Severity = anomaly.Severity
                });
        }
    }
}

// Consumer group for scaling
[ConsumerGroup("sensor-processors", Size = 10)]
[MessageHandler(PrefetchCount = 100)]
public class SensorDataProcessor : 
    IMessageHandler<SensorReading>
{
    private readonly IBatchCollector _collector;
    
    public async Task HandleAsync(
        SensorReading reading, 
        MessageContext context)
    {
        // Collect readings into batches
        await _collector.AddAsync(reading);
        
        // Batch is automatically flushed 
        // when size or time threshold is reached
    }
}
```

</td>
<td>

```go
// Batch processing for high throughput
type SensorDataBatchHandler struct {
    tsdb      TimeSeriesDatabase
    analytics AnalyticsEngine
    publisher messaging.Publisher
}

func (h *SensorDataBatchHandler) Handle(
    ctx context.Context, 
    msg contracts.Message) error {
    
    batch := msg.(*SensorDataBatch)
    
    // Validate batch
    validReadings := make([]SensorReading, 0, len(batch.Readings))
    for _, r := range batch.Readings {
        if r.IsValid() {
            validReadings = append(validReadings, r)
        }
    }
    
    // Store in time-series database
    if err := h.tsdb.WriteBatch(ctx, validReadings); err != nil {
        return err
    }
    
    // Real-time analytics
    anomalies, err := h.analytics.DetectAnomalies(
        ctx, validReadings)
    if err != nil {
        return err
    }
    
    // Publish alerts for anomalies
    for _, anomaly := range anomalies {
        event := &AnomalyDetectedEvent{
            BaseEvent:   contracts.NewBaseEvent("AnomalyDetectedEvent", anomaly.DeviceID),
            DeviceID:    anomaly.DeviceID,
            ReadingType: anomaly.Type,
            Value:       anomaly.Value,
            Threshold:   anomaly.Threshold,
            Severity:    anomaly.Severity,
        }
        
        if err := h.publisher.PublishEvent(ctx, event); err != nil {
            log.Printf("Failed to publish anomaly: %v", err)
        }
    }
    
    return nil
}

// Consumer group for scaling
func StartSensorProcessors(
    subscriber messaging.Subscriber,
    collector BatchCollector) error {
    
    group := messaging.NewConsumerGroup(
        "sensor-processors",
        messaging.WithGroupSize(10),
        messaging.WithPrefetchCount(100))
    
    handler := &SensorDataProcessor{
        collector: collector,
    }
    
    return group.Start(context.Background(), 
        subscriber, "sensor.readings", handler)
}

type SensorDataProcessor struct {
    collector BatchCollector
}

func (p *SensorDataProcessor) Handle(
    ctx context.Context, 
    msg contracts.Message) error {
    
    reading := msg.(*SensorReading)
    
    // Collect readings into batches
    return p.collector.Add(ctx, reading)
    // Batch is automatically flushed 
    // when size or time threshold is reached
}
```

</td>
</tr>
</table>

## Best Practices from Real Implementations

### 1. Error Handling Strategy
- Use specific exception/error types for different failure modes
- Implement circuit breakers for external service calls
- Use dead letter queues for unprocessable messages
- Log errors with full context for debugging

### 2. Performance Optimization
- Batch operations when dealing with high volume
- Use consumer groups for horizontal scaling
- Implement caching for frequently accessed data
- Monitor queue depths and adjust concurrency

### 3. Security Considerations
- Encrypt sensitive data in messages
- Use message-level authentication for critical operations
- Implement audit logging for compliance
- Apply principle of least privilege for queue access

### 4. Monitoring and Observability
- Track message processing metrics
- Monitor workflow completion rates
- Set up alerts for anomalies
- Use distributed tracing for debugging

### 5. Testing Strategies
- Use in-memory transport for unit tests
- Create integration tests with real brokers
- Test compensation logic thoroughly
- Simulate failure scenarios

## Deployment Patterns

### Blue-Green Deployment
Deploy new versions alongside old ones, using message versioning to ensure compatibility during transition.

### Canary Releases
Route a percentage of messages to new handlers while monitoring for issues.

### Feature Toggles
Use message headers or configuration to enable/disable features without deployment.

### Rolling Updates
Update consumers gradually while maintaining service availability.

## Conclusion

These complete solutions demonstrate how Mmate's components work together to solve complex business problems. Key takeaways:

1. **Modular Architecture**: Each component has a specific responsibility
2. **Reliability First**: Use workflows with compensation for critical operations
3. **Scalability**: Design for horizontal scaling from the start
4. **Observability**: Instrument everything for production debugging
5. **Security**: Build security into the message pipeline, not as an afterthought