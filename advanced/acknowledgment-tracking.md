# Application-Level Acknowledgment Tracking

> **⚠️ Go-Only Feature**: Application-level acknowledgment tracking is only available in the Go implementation of Mmate. The .NET implementation provides basic message acknowledgments through RabbitMQ only.

Application-level acknowledgment tracking provides end-to-end visibility into message processing status. Unlike transport-level acknowledgments (which only confirm message delivery), application acknowledgments confirm that business logic has successfully processed the message.

## How It Works

The acknowledgment tracking system uses correlation IDs to match requests with processing confirmations:

1. **Message Publishing**: Messages are sent with `reply-to` headers pointing to an acknowledgment queue
2. **Correlation Tracking**: Each message gets a unique correlation ID for tracking
3. **Processing Acknowledgment**: Handlers automatically send acknowledgment messages upon completion
4. **Response Matching**: Acknowledgments are matched to original requests using correlation IDs
5. **Timeout Handling**: Unacknowledged messages are automatically cleaned up after timeout

## Key Benefits

- **End-to-End Visibility**: Track message processing from send to completion
- **Business Logic Confirmation**: Verify that business processing succeeded, not just delivery
- **Correlation Management**: Automatic correlation ID handling and response matching
- **Timeout Protection**: Automatic cleanup of stale acknowledgment tracking
- **Error Reporting**: Detailed error information when processing fails

## Basic Configuration

```go
import (
    "github.com/glimte/mmate-go"
    "time"
)

// Create client with acknowledgment tracking enabled
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithAcknowledgmentTracking(30*time.Second), // timeout
    mmate.WithServiceName("order-service"),
)
if err != nil {
    log.Fatal("Failed to create client:", err)
}
```

## Sending Messages with Acknowledgment

### Basic Send with Acknowledgment

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderService
{
    private readonly IMessageSender _sender;
    
    public async Task<ProcessingAcknowledgment> CreateOrderAsync(
        CreateOrderCommand command)
    {
        // Send command and wait for processing acknowledgment
        var ackResponse = await _sender.SendWithAckAsync(
            command, 
            "orders-queue",
            timeout: TimeSpan.FromSeconds(60));
        
        // Wait for acknowledgment
        var ack = await ackResponse.WaitForAcknowledgmentAsync();
        
        if (ack.Success)
        {
            Console.WriteLine($"Order processed in {ack.ProcessingTime}");
            return ack;
        }
        else
        {
            throw new Exception($"Order processing failed: {ack.ErrorMessage}");
        }
    }
}
```

</td>
<td>

```go
func (s *OrderService) CreateOrder(ctx context.Context, command *CreateOrderCommand) error {
    // Send command and get acknowledgment response
    ackResponse, err := s.client.SendWithAck(ctx, command)
    if err != nil {
        return fmt.Errorf("failed to send command: %w", err)
    }
    
    // Wait for processing acknowledgment
    ack, err := ackResponse.WaitForAcknowledgment(ctx)
    if err != nil {
        return fmt.Errorf("acknowledgment timeout: %w", err)
    }
    
    if ack.Success {
        log.Printf("Order processed in %v", ack.ProcessingTime)
        return nil
    } else {
        return fmt.Errorf("order processing failed: %s", ack.ErrorMessage)
    }
}
```

</td>
</tr>
</table>

### Batch Operations with Acknowledgment

<table>
<tr>
<th>.NET</th>
<th>Go</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task<List<ProcessingAcknowledgment>> ProcessMultipleOrdersAsync(
    List<CreateOrderCommand> commands)
{
    var ackTasks = new List<Task<ProcessingAcknowledgment>>();
    
    // Send all commands
    foreach (var command in commands)
    {
        var ackResponse = await _sender.SendWithAckAsync(
            command, "orders-queue");
        ackTasks.Add(ackResponse.WaitForAcknowledgmentAsync());
    }
    
    // Wait for all acknowledgments
    var acknowledgments = await Task.WhenAll(ackTasks);
    
    // Process results
    var successful = acknowledgments.Count(a => a.Success);
    var failed = acknowledgments.Count(a => !a.Success);
    
    Console.WriteLine($"Processed: {successful} successful, {failed} failed");
    return acknowledgments.ToList();
}
```

</td>
<td>

```go
func (s *OrderService) ProcessMultipleOrders(
    ctx context.Context, 
    commands []*CreateOrderCommand) ([]*messaging.ProcessingAcknowledgment, error) {
    
    var ackResponses []*messaging.AckResponse
    
    // Send all commands
    for _, command := range commands {
        ackResponse, err := s.client.SendWithAck(ctx, command)
        if err != nil {
            return nil, fmt.Errorf("failed to send command: %w", err)
        }
        ackResponses = append(ackResponses, ackResponse)
    }
    
    // Wait for all acknowledgments
    var acknowledgments []*messaging.ProcessingAcknowledgment
    var successful, failed int
    
    for _, ackResponse := range ackResponses {
        ack, err := ackResponse.WaitForAcknowledgment(ctx)
        if err != nil {
            log.Printf("Acknowledgment timeout: %v", err)
            failed++
            continue
        }
        
        acknowledgments = append(acknowledgments, ack)
        if ack.Success {
            successful++
        } else {
            failed++
        }
    }
    
    log.Printf("Processed: %d successful, %d failed", successful, failed)
    return acknowledgments, nil
}
```

</td>
</tr>
</table>

## Event Publishing with Acknowledgment

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task PublishOrderCreatedEventAsync(OrderCreatedEvent evt)
{
    // Publish event and track acknowledgment
    var ackResponse = await _publisher.PublishWithAckAsync(evt);
    
    // Optional: Wait for acknowledgment or handle async
    _ = Task.Run(async () =>
    {
        try
        {
            var ack = await ackResponse.WaitForAcknowledgmentAsync(
                timeout: TimeSpan.FromSeconds(10));
            
            if (ack.Success)
            {
                _metrics.Increment("events.published.confirmed");
            }
            else
            {
                _metrics.Increment("events.published.failed");
                _logger.LogWarning("Event processing failed: {Error}", 
                    ack.ErrorMessage);
            }
        }
        catch (TimeoutException)
        {
            _metrics.Increment("events.published.timeout");
        }
    });
}
```

</td>
<td>

```go
func (s *OrderService) PublishOrderCreatedEvent(
    ctx context.Context, 
    evt *OrderCreatedEvent) error {
    
    // Publish event with acknowledgment tracking
    ackResponse, err := s.client.PublishEventWithAck(ctx, evt)
    if err != nil {
        return fmt.Errorf("failed to publish event: %w", err)
    }
    
    // Handle acknowledgment asynchronously
    go func() {
        ackCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        
        ack, err := ackResponse.WaitForAcknowledgment(ackCtx)
        if err != nil {
            s.metrics.Increment("events.published.timeout")
            log.Printf("Event acknowledgment timeout: %v", err)
            return
        }
        
        if ack.Success {
            s.metrics.Increment("events.published.confirmed")
        } else {
            s.metrics.Increment("events.published.failed")
            log.Printf("Event processing failed: %s", ack.ErrorMessage)
        }
    }()
    
    return nil
}
```

</td>
</tr>
</table>

## Automatic Processing Acknowledgment

### Handler-Level Auto-Acknowledgment

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[MessageHandler("orders-queue")]
[AutoAcknowledgment] // Automatic acknowledgment interceptor
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    public async Task HandleAsync(
        CreateOrderCommand command, 
        CancellationToken ct)
    {
        var startTime = DateTime.UtcNow;
        
        try
        {
            // Process the order
            var order = await _orderService.CreateOrderAsync(command);
            
            // Success acknowledgment sent automatically
            // Includes processing time and success status
        }
        catch (Exception ex)
        {
            // Error acknowledgment sent automatically
            // Includes error message and processing time
            throw;
        }
    }
}
```

</td>
<td>

```go
// Acknowledgment tracking is automatically enabled when configured
// Processing acknowledgments are sent automatically by interceptor

func (h *OrderHandler) HandleCreateOrder(
    ctx context.Context, 
    msg contracts.Message) error {
    
    command := msg.(*CreateOrderCommand)
    
    // Process the order
    order, err := h.orderService.CreateOrder(ctx, command)
    if err != nil {
        // Error acknowledgment sent automatically with error details
        return fmt.Errorf("failed to create order: %w", err)
    }
    
    // Success acknowledgment sent automatically with processing time
    log.Printf("Order created: %s", order.ID)
    return nil
}

// Subscribe with automatic acknowledgment
err = client.Subscriber().Subscribe(
    ctx,
    "orders-queue", 
    "CreateOrderCommand",
    h.HandleCreateOrder, // Auto-ack interceptor wraps this handler
)
```

</td>
</tr>
</table>

## Custom Acknowledgment Messages

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task HandleAsync(
    ProcessPaymentCommand command, 
    CancellationToken ct)
{
    var result = await _paymentService.ProcessAsync(command);
    
    // Send custom acknowledgment with additional metadata
    var ack = new ProcessingAcknowledgment
    {
        CorrelationID = command.CorrelationID,
        MessageID = command.ID,
        Success = result.Success,
        ProcessingTime = result.Duration,
        Metadata = new Dictionary<string, object>
        {
            ["TransactionId"] = result.TransactionId,
            ["PaymentMethod"] = command.PaymentMethod,
            ["Amount"] = command.Amount,
            ["Currency"] = command.Currency
        }
    };
    
    if (!result.Success)
    {
        ack.ErrorMessage = result.ErrorMessage;
        ack.Metadata["ErrorCode"] = result.ErrorCode;
    }
    
    await _ackService.SendAcknowledgmentAsync(ack);
}
```

</td>
<td>

```go
func (h *PaymentHandler) HandleProcessPayment(
    ctx context.Context, 
    msg contracts.Message) error {
    
    command := msg.(*ProcessPaymentCommand)
    result, err := h.paymentService.Process(ctx, command)
    
    // Create custom acknowledgment
    ack := &messaging.ProcessingAcknowledgment{
        CorrelationID:  command.GetCorrelationID(),
        MessageID:      command.GetID(),
        MessageType:    command.GetType(),
        Success:        err == nil,
        ProcessingTime: result.Duration,
        ProcessedAt:    time.Now(),
        ProcessorID:    h.processorID,
        Metadata: map[string]interface{}{
            "TransactionId":  result.TransactionId,
            "PaymentMethod":  command.PaymentMethod,
            "Amount":         command.Amount,
            "Currency":       command.Currency,
        },
    }
    
    if err != nil {
        ack.ErrorMessage = err.Error()
        ack.Metadata["ErrorCode"] = result.ErrorCode
    }
    
    // Send custom acknowledgment
    tracker := h.client.AcknowledgmentTracker()
    return tracker.HandleAcknowledgment(ctx, 
        messaging.NewAcknowledgmentMessage(
            ack.CorrelationID, ack.MessageID, ack.MessageType, ack.Success))
}
```

</td>
</tr>
</table>

## Monitoring and Metrics

### Acknowledgment Metrics

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class AcknowledgmentMetricsInterceptor : MessageInterceptorBase
{
    private readonly IMetrics _metrics;
    
    public override async Task OnAcknowledgmentReceivedAsync(
        ProcessingAcknowledgment ack)
    {
        var tags = new[]
        {
            ("message_type", ack.MessageType),
            ("success", ack.Success.ToString()),
            ("processor_id", ack.ProcessorID)
        };
        
        _metrics.Increment("acknowledgment.received", tags);
        _metrics.Histogram("acknowledgment.processing_time", 
            ack.ProcessingTime.TotalMilliseconds, tags);
        
        if (!ack.Success)
        {
            _metrics.Increment("acknowledgment.errors", 
                tags.Concat(new[] { ("error", ack.ErrorMessage) }));
        }
    }
    
    public override async Task OnAcknowledgmentTimeoutAsync(
        string correlationId, 
        string messageType)
    {
        _metrics.Increment("acknowledgment.timeout", new[]
        {
            ("message_type", messageType),
            ("correlation_id", correlationId)
        });
    }
}
```

</td>
<td>

```go
type AcknowledgmentMetrics struct {
    metrics metrics.Client
}

func (m *AcknowledgmentMetrics) OnAcknowledgmentReceived(
    ack *messaging.ProcessingAcknowledgment) {
    
    tags := metrics.Tags{
        "message_type": ack.MessageType,
        "success":      fmt.Sprintf("%t", ack.Success),
        "processor_id": ack.ProcessorID,
    }
    
    m.metrics.Increment("acknowledgment.received", tags)
    m.metrics.Histogram("acknowledgment.processing_time", 
        float64(ack.ProcessingTime.Milliseconds()), tags)
    
    if !ack.Success {
        errorTags := maps.Clone(tags)
        errorTags["error"] = ack.ErrorMessage
        m.metrics.Increment("acknowledgment.errors", errorTags)
    }
}

func (m *AcknowledgmentMetrics) OnAcknowledgmentTimeout(
    correlationID, messageType string) {
    
    m.metrics.Increment("acknowledgment.timeout", metrics.Tags{
        "message_type":   messageType,
        "correlation_id": correlationID,
    })
}

// Integration with acknowledgment tracker
tracker := client.AcknowledgmentTracker()
metrics := &AcknowledgmentMetrics{metrics: metricsClient}

// Monitor acknowledgment events
go func() {
    for {
        select {
        case ack := <-tracker.AcknowledgmentReceived():
            metrics.OnAcknowledgmentReceived(ack)
        case timeout := <-tracker.AcknowledgmentTimeout():
            metrics.OnAcknowledgmentTimeout(timeout.CorrelationID, timeout.MessageType)
        }
    }
}()
```

</td>
</tr>
</table>

## Testing Acknowledgment Tracking

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task SendWithAck_ShouldReceiveSuccessAcknowledgment()
{
    // Arrange
    var command = new CreateOrderCommand { CustomerId = "123" };
    var mockHandler = new Mock<IMessageHandler<CreateOrderCommand>>();
    mockHandler.Setup(h => h.HandleAsync(It.IsAny<CreateOrderCommand>(), 
        It.IsAny<CancellationToken>()))
        .Returns(Task.CompletedTask);
    
    // Setup test handler
    _testHarness.RegisterHandler("orders-queue", mockHandler.Object);
    
    // Act
    var ackResponse = await _sender.SendWithAckAsync(command, "orders-queue");
    var ack = await ackResponse.WaitForAcknowledgmentAsync(
        timeout: TimeSpan.FromSeconds(5));
    
    // Assert
    Assert.IsTrue(ack.Success);
    Assert.AreEqual(command.ID, ack.MessageID);
    Assert.AreEqual(command.CorrelationID, ack.CorrelationID);
    Assert.IsTrue(ack.ProcessingTime > TimeSpan.Zero);
    mockHandler.Verify(h => h.HandleAsync(command, 
        It.IsAny<CancellationToken>()), Times.Once);
}

[Test]
public async Task SendWithAck_ShouldReceiveErrorAcknowledgment()
{
    // Arrange
    var command = new CreateOrderCommand { CustomerId = "invalid" };
    var mockHandler = new Mock<IMessageHandler<CreateOrderCommand>>();
    mockHandler.Setup(h => h.HandleAsync(It.IsAny<CreateOrderCommand>(), 
        It.IsAny<CancellationToken>()))
        .ThrowsAsync(new ArgumentException("Invalid customer"));
    
    _testHarness.RegisterHandler("orders-queue", mockHandler.Object);
    
    // Act
    var ackResponse = await _sender.SendWithAckAsync(command, "orders-queue");
    var ack = await ackResponse.WaitForAcknowledgmentAsync();
    
    // Assert
    Assert.IsFalse(ack.Success);
    Assert.AreEqual("Invalid customer", ack.ErrorMessage);
    Assert.IsTrue(ack.ProcessingTime > TimeSpan.Zero);
}
```

</td>
<td>

```go
func TestSendWithAck_ShouldReceiveSuccessAcknowledgment(t *testing.T) {
    // Arrange
    client := setupTestClient(t)
    command := &CreateOrderCommand{CustomerID: "123"}
    
    // Setup test handler that succeeds
    err := client.Subscriber().Subscribe(
        context.Background(),
        "orders-queue", 
        "CreateOrderCommand",
        func(ctx context.Context, msg contracts.Message) error {
            // Simulate successful processing
            time.Sleep(10 * time.Millisecond)
            return nil
        },
    )
    require.NoError(t, err)
    
    // Act
    ackResponse, err := client.SendWithAck(context.Background(), command)
    require.NoError(t, err)
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    ack, err := ackResponse.WaitForAcknowledgment(ctx)
    
    // Assert
    require.NoError(t, err)
    assert.True(t, ack.Success)
    assert.Equal(t, command.GetID(), ack.MessageID)
    assert.Equal(t, command.GetCorrelationID(), ack.CorrelationID)
    assert.Greater(t, ack.ProcessingTime, time.Duration(0))
}

func TestSendWithAck_ShouldReceiveErrorAcknowledgment(t *testing.T) {
    // Arrange
    client := setupTestClient(t)
    command := &CreateOrderCommand{CustomerID: "invalid"}
    
    // Setup test handler that fails
    err := client.Subscriber().Subscribe(
        context.Background(),
        "orders-queue", 
        "CreateOrderCommand",
        func(ctx context.Context, msg contracts.Message) error {
            return errors.New("invalid customer")
        },
    )
    require.NoError(t, err)
    
    // Act
    ackResponse, err := client.SendWithAck(context.Background(), command)
    require.NoError(t, err)
    
    ack, err := ackResponse.WaitForAcknowledgment(context.Background())
    
    // Assert
    require.NoError(t, err)
    assert.False(t, ack.Success)
    assert.Contains(t, ack.ErrorMessage, "invalid customer")
    assert.Greater(t, ack.ProcessingTime, time.Duration(0))
}
```

</td>
</tr>
</table>

## Best Practices

1. **Use Appropriate Timeouts**
   - Short operations: 5-30 seconds
   - Long operations: 1-5 minutes
   - Batch operations: Scale with batch size

2. **Handle Acknowledgment Errors**
   - Always check acknowledgment success status
   - Log failed acknowledgments for investigation
   - Implement fallback strategies for failures

3. **Monitor Acknowledgment Rates**
   - Track acknowledgment success/failure rates
   - Monitor acknowledgment timeout rates
   - Alert on abnormal patterns

4. **Correlation ID Management**
   - Use meaningful correlation IDs for tracing
   - Include request context in correlation IDs
   - Ensure correlation IDs are unique and traceable

5. **Performance Considerations**
   - Use acknowledgment tracking for critical operations
   - Consider fire-and-forget for high-volume, non-critical messages
   - Batch acknowledgment requests when possible

## Troubleshooting

### Common Issues

**Acknowledgment Timeouts**
- Check handler processing time vs timeout settings
- Verify acknowledgment queue configuration
- Monitor handler error rates

**Missing Acknowledgments**
- Verify reply-to header configuration
- Check acknowledgment queue bindings
- Ensure processing acknowledgment interceptor is enabled

**Memory Leaks**
- Monitor pending acknowledgment count
- Verify timeout cleanup is working
- Check for correlation ID collisions

### Debugging

```go
// Enable detailed acknowledgment logging
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithAcknowledgmentTracking(30*time.Second),
    mmate.WithLogger(logger),
)

// Monitor acknowledgment tracker state
tracker := client.AcknowledgmentTracker()
log.Printf("Pending acknowledgments: %d", tracker.GetPendingAcksCount())
```