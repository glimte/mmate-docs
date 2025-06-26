# Auto-Acknowledgment Feature

The Auto-Acknowledgment feature allows message handlers to automatically send acknowledgment messages back to the sender during message processing. This provides a way for senders to track the status and outcome of their requests in asynchronous messaging scenarios.

## Overview

Auto-acknowledgment enables:
- Real-time visibility into message processing status
- Reliable tracking of message receipt and processing
- Standardized acknowledgment patterns across handlers
- Clear error indication when processing fails
- Better monitoring and observability

## Sending Messages with Acknowledgment

Send messages that expect acknowledgment responses:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Send message and get acknowledgment tracker
var response = await dispatcher.SendWithAckAsync(
    new ProcessOrderCommand 
    { 
        OrderId = "ORDER-123",
        CustomerId = "CUST-456"
    });

// Wait for acknowledgment
var ack = await response.WaitForAcknowledgmentAsync();
if (ack.Success)
{
    logger.LogInformation("Order processing acknowledged");
}
```

</td>
<td>

```go
// Send message and get acknowledgment tracker
response, err := dispatcher.SendWithAck(ctx, 
    &ProcessOrderCommand{
        OrderID:    "ORDER-123",
        CustomerID: "CUST-456",
    })
if err != nil {
    return err
}

// Wait for acknowledgment
ack, err := response.WaitForAcknowledgment(ctx)
if err != nil {
    return err
}

if ack.Success {
    log.Info("Order processing acknowledged")
}
```

</td>
</tr>
</table>

## Creating Auto-Acknowledging Handlers

Create handlers that automatically send acknowledgments:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ProcessOrderHandler : 
    AutoAcknowledgingMessageHandler<ProcessOrderCommand>
{
    private readonly IOrderService _orderService;
    
    public ProcessOrderHandler(
        IMessageDispatcher messageDispatcher,
        ILogger<ProcessOrderHandler> logger,
        IOrderService orderService) 
        : base(messageDispatcher, logger)
    {
        _orderService = orderService;
    }

    protected override async Task HandleMessageAsync(
        ProcessOrderCommand message, 
        IMessageContext context)
    {
        // Process the message
        await _orderService.ProcessOrder(
            message.OrderId, message.CustomerId);
        
        // Acknowledgment is automatically sent upon completion
    }
}

// Register the handler
services.AddMessageHandler<ProcessOrderHandler, ProcessOrderCommand>();
```

</td>
<td>

```go
type ProcessOrderHandler struct {
    orderService OrderService
    ackSender    messaging.AckSender
}

func (h *ProcessOrderHandler) Handle(
    ctx context.Context, 
    cmd *ProcessOrderCommand) error {
    
    // Process the message
    err := h.orderService.ProcessOrder(ctx, 
        cmd.OrderID, cmd.CustomerID)
    
    // Send acknowledgment based on result
    ack := &messaging.Acknowledgment{
        CorrelationID: cmd.CorrelationID,
        Success:       err == nil,
        Timestamp:     time.Now(),
    }
    
    if err != nil {
        ack.ErrorMessage = err.Error()
    }
    
    return h.ackSender.SendAck(ctx, ack, cmd.ReplyTo)
}

// Register handler with auto-acknowledgment
dispatcher.HandleWithAck("process.order", &ProcessOrderHandler{
    orderService: orderService,
    ackSender:    ackSender,
})
```

</td>
</tr>
</table>

## Checking Acknowledgment Status

Monitor acknowledgment status after sending messages:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Send with acknowledgment
var response = await dispatcher.SendWithAckAsync(command);

// Check if acknowledged (non-blocking)
if (await response.IsAcknowledgedAsync())
{
    logger.LogInformation("Message already acknowledged");
}

// Wait for acknowledgment with timeout
try
{
    var ack = await response.WaitForAcknowledgmentAsync(
        TimeSpan.FromSeconds(30));
    
    if (ack.Success)
    {
        logger.LogInformation("Processing succeeded");
    }
    else
    {
        logger.LogError("Processing failed: {Error}", 
            ack.ErrorMessage);
    }
}
catch (TimeoutException)
{
    logger.LogWarning("Acknowledgment timeout");
}
```

</td>
<td>

```go
// Send with acknowledgment
response, err := dispatcher.SendWithAck(ctx, command)
if err != nil {
    return err
}

// Check if acknowledged (non-blocking)
if response.IsAcknowledged() {
    log.Info("Message already acknowledged")
}

// Wait for acknowledgment with timeout
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()

ack, err := response.WaitForAcknowledgment(ctx)
if err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        log.Warn("Acknowledgment timeout")
        return err
    }
    return err
}

if ack.Success {
    log.Info("Processing succeeded")
} else {
    log.Error("Processing failed", "error", ack.ErrorMessage)
}
```

</td>
</tr>
</table>

## Correlation IDs

The auto-acknowledgment system uses correlation IDs to match responses:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Correlation ID is automatically generated
var response = await dispatcher.SendWithAckAsync(
    new ProcessOrderCommand { OrderId = "ORDER-123" });

// Access the correlation ID
string correlationId = response.CorrelationId;
logger.LogInformation("Sent message with correlation ID: {Id}", 
    correlationId);

// The acknowledgment will include the same correlation ID
var ack = await response.WaitForAcknowledgmentAsync();
Assert.AreEqual(correlationId, ack.CorrelationId);
```

</td>
<td>

```go
// Correlation ID is automatically generated
response, err := dispatcher.SendWithAck(ctx, 
    &ProcessOrderCommand{OrderID: "ORDER-123"})
if err != nil {
    return err
}

// Access the correlation ID
correlationID := response.CorrelationID()
log.Info("Sent message with correlation ID", 
    "id", correlationID)

// The acknowledgment will include the same correlation ID
ack, err := response.WaitForAcknowledgment(ctx)
if err != nil {
    return err
}
assert.Equal(t, correlationID, ack.CorrelationID)
```

</td>
</tr>
</table>

## Error Handling

Handle errors in acknowledgments gracefully:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderHandler : 
    AutoAcknowledgingMessageHandler<ProcessOrderCommand>
{
    protected override async Task HandleMessageAsync(
        ProcessOrderCommand message, 
        IMessageContext context)
    {
        try
        {
            // Validate message
            if (string.IsNullOrEmpty(message.OrderId))
            {
                // Send custom error acknowledgment
                await SendErrorAcknowledgmentAsync(
                    message, context, "OrderId is required");
                return;
            }
            
            // Process order
            await ProcessOrder(message);
            
            // Success acknowledgment sent automatically
        }
        catch (ValidationException ex)
        {
            // Send error acknowledgment
            await SendErrorAcknowledgmentAsync(
                message, context, ex.Message);
        }
        catch (Exception ex)
        {
            // Let base class handle unexpected errors
            throw;
        }
    }
}
```

</td>
<td>

```go
func (h *OrderHandler) Handle(
    ctx context.Context, 
    cmd *ProcessOrderCommand) error {
    
    ack := &messaging.Acknowledgment{
        CorrelationID: cmd.CorrelationID,
        Timestamp:     time.Now(),
    }
    
    defer func() {
        h.ackSender.SendAck(ctx, ack, cmd.ReplyTo)
    }()
    
    // Validate message
    if cmd.OrderID == "" {
        ack.Success = false
        ack.ErrorMessage = "OrderID is required"
        return nil // Don't return error - ack will indicate failure
    }
    
    // Process order
    err := h.processOrder(ctx, cmd)
    if err != nil {
        ack.Success = false
        ack.ErrorMessage = err.Error()
        
        // Check if it's a validation error vs system error
        var validationErr *ValidationError
        if errors.As(err, &validationErr) {
            return nil // Don't retry validation errors
        }
        
        return err // System errors should be retried
    }
    
    ack.Success = true
    return nil
}
```

</td>
</tr>
</table>

## Advanced Configuration

### Timeout Configuration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure global acknowledgment timeout
services.AddMmateMessaging(options =>
{
    options.DefaultAckTimeout = TimeSpan.FromSeconds(30);
});

// Per-request timeout
var ack = await response.WaitForAcknowledgmentAsync(
    TimeSpan.FromMinutes(2));
```

</td>
<td>

```go
// Configure global acknowledgment timeout
config := messaging.Config{
    DefaultAckTimeout: 30 * time.Second,
}

// Per-request timeout
ctx, cancel := context.WithTimeout(ctx, 2*time.Minute)
defer cancel()

ack, err := response.WaitForAcknowledgment(ctx)
```

</td>
</tr>
</table>

### Custom Acknowledgment Data

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
protected override async Task SendAcknowledgmentAsync(
    string correlationId,
    string replyToQueue,
    AckStatus status,
    string message = null)
{
    var ack = new Acknowledgment
    {
        CorrelationId = correlationId,
        Status = status,
        Message = message,
        OriginalMessageType = typeof(ProcessOrderCommand).FullName,
        Timestamp = DateTimeOffset.UtcNow,
        
        // Add custom data
        ProcessingTime = _stopwatch.Elapsed,
        WorkerId = Environment.MachineName,
        EstimatedCompletionTime = status == AckStatus.Processing 
            ? TimeSpan.FromMinutes(5) 
            : null
    };
    
    await _dispatcher.SendAcknowledgmentAsync(ack, replyToQueue);
}
```

</td>
<td>

```go
func (h *Handler) sendCustomAck(
    ctx context.Context,
    correlationID string,
    replyTo string,
    success bool,
    err error) error {
    
    ack := &messaging.Acknowledgment{
        CorrelationID: correlationID,
        Success:       success,
        Timestamp:     time.Now(),
        
        // Add custom data
        ProcessingTime: time.Since(h.startTime),
        WorkerID:       os.Getenv("HOSTNAME"),
        MessageType:    "ProcessOrderCommand",
    }
    
    if err != nil {
        ack.ErrorMessage = err.Error()
    }
    
    if !success && ack.ErrorMessage == "" {
        ack.EstimatedCompletionTime = 5 * time.Minute
    }
    
    return h.ackSender.SendAck(ctx, ack, replyTo)
}
```

</td>
</tr>
</table>

## Requirements for Automatic Acknowledgments

For automatic acknowledgments to work properly:

1. **Message Structure**: Messages must include:
   - `CorrelationId` field for request tracking
   - `ReplyTo` field specifying acknowledgment queue

2. **Sender Configuration**: 
   - Use `SendWithAck` methods
   - Specify acknowledgment queue
   - Handle timeout scenarios

3. **Handler Implementation**:
   - Extend auto-acknowledging base classes or implement ack sending
   - Handle errors appropriately
   - Send acknowledgments even on failures

## Benefits

- **Real-time Visibility**: Immediate feedback on message processing status
- **Reliability**: Track if messages are received and processed successfully
- **Standardization**: Consistent acknowledgment patterns across all handlers
- **Error Transparency**: Clear indication when and why processing fails
- **Monitoring**: Better insight into message processing performance and bottlenecks
- **Debugging**: Easier troubleshooting with correlation IDs and detailed error messages

## Best Practices

1. **Always Handle Timeouts**: Set appropriate timeouts for acknowledgment waits
2. **Distinguish Error Types**: Different handling for validation vs system errors
3. **Include Correlation Context**: Use correlation IDs for distributed tracing
4. **Monitor Ack Rates**: Track acknowledgment success/failure rates
5. **Graceful Degradation**: Handle missing acknowledgments appropriately
6. **Custom Error Details**: Provide meaningful error messages in acknowledgments