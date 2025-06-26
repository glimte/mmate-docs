# Testing Strategies

This guide covers testing approaches for Mmate applications, from unit tests to end-to-end integration testing.

## Unit Testing Message Handlers

### Basic Handler Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestClass]
public class OrderHandlerTests
{
    [TestMethod]
    public async Task HandleAsync_ValidOrder_CreatesOrder()
    {
        // Arrange
        var mockRepository = new Mock<IOrderRepository>();
        var mockPublisher = new Mock<IMessagePublisher>();
        var handler = new CreateOrderHandler(
            mockRepository.Object, 
            mockPublisher.Object);
        
        var command = new CreateOrderCommand
        {
            CustomerId = "CUST-123",
            Items = new[] { new OrderItem { ProductId = "P1" } }
        };
        
        // Act
        await handler.HandleAsync(command, CancellationToken.None);
        
        // Assert
        mockRepository.Verify(r => r.SaveAsync(
            It.IsAny<Order>(), 
            It.IsAny<CancellationToken>()), Times.Once);
        
        mockPublisher.Verify(p => p.PublishAsync(
            It.Is<OrderCreatedEvent>(e => e.CustomerId == "CUST-123"),
            It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

</td>
<td>

```go
func TestOrderHandler_Handle_ValidOrder_CreatesOrder(t *testing.T) {
    // Arrange
    mockRepo := &MockOrderRepository{}
    mockPublisher := &MockMessagePublisher{}
    handler := &CreateOrderHandler{
        repository: mockRepo,
        publisher:  mockPublisher,
    }
    
    command := &CreateOrderCommand{
        CustomerID: "CUST-123",
        Items: []OrderItem{
            {ProductID: "P1"},
        },
    }
    
    // Act
    err := handler.Handle(context.Background(), command)
    
    // Assert
    assert.NoError(t, err)
    assert.True(t, mockRepo.SaveCalled)
    assert.True(t, mockPublisher.PublishCalled)
    assert.Equal(t, "CUST-123", 
        mockPublisher.PublishedEvent.(*OrderCreatedEvent).CustomerID)
}

type MockOrderRepository struct {
    SaveCalled bool
}

func (m *MockOrderRepository) Save(
    ctx context.Context, 
    order *Order) error {
    m.SaveCalled = true
    return nil
}
```

</td>
</tr>
</table>

### Testing Error Handling

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestMethod]
public async Task HandleAsync_DatabaseError_ThrowsException()
{
    // Arrange
    var mockRepository = new Mock<IOrderRepository>();
    mockRepository
        .Setup(r => r.SaveAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
        .ThrowsAsync(new DatabaseException("Connection failed"));
    
    var handler = new CreateOrderHandler(mockRepository.Object, null);
    var command = new CreateOrderCommand { CustomerId = "CUST-123" };
    
    // Act & Assert
    await Assert.ThrowsExceptionAsync<DatabaseException>(() =>
        handler.HandleAsync(command, CancellationToken.None));
}
```

</td>
<td>

```go
func TestOrderHandler_Handle_DatabaseError_ReturnsError(t *testing.T) {
    // Arrange
    mockRepo := &MockOrderRepository{
        ShouldError: true,
        Error: errors.New("connection failed"),
    }
    handler := &CreateOrderHandler{repository: mockRepo}
    command := &CreateOrderCommand{CustomerID: "CUST-123"}
    
    // Act
    err := handler.Handle(context.Background(), command)
    
    // Assert
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "connection failed")
}
```

</td>
</tr>
</table>

## Testing Interceptors

### Interceptor Unit Tests

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestClass]
public class InterceptorTests
{
    [TestMethod]
    public async Task RetryInterceptor_ShouldRetryOnTransientError()
    {
        // Arrange
        var services = new ServiceCollection();
        services.AddLogging();
        services.AddMmateInterceptors()
            .ConfigureRetryPolicies(options =>
            {
                options.GlobalMaxRetries = 3;
                options.UseExponentialBackoff = false;
            });
            
        var provider = services.BuildServiceProvider();
        var interceptor = provider.GetRequiredService<RetryInterceptor>();
        
        var context = MessageContext.CreateForTesting();
        var message = new TestMessage();
        
        // Simulate a transient error
        context.SetException(new TimeoutException("Test timeout"));
        
        // Act
        await interceptor.OnAfterConsumeAsync(context, message, false);
        
        // Assert
        Assert.IsTrue(context.GetHeader<int>("RetryAttempt") > 0);
        Assert.AreEqual("Timeout", context.GetHeader<string>("ErrorType"));
    }
}
```

</td>
<td>

```go
func TestRetryInterceptor_ShouldRetryOnTransientError(t *testing.T) {
    // Arrange
    interceptor := &RetryInterceptor{
        maxRetries: 3,
        backoff:    &FixedBackoff{delay: time.Millisecond},
    }
    
    ctx := context.Background()
    msg := &TestMessage{}
    
    attempts := 0
    handler := func(ctx context.Context, msg messaging.Message) error {
        attempts++
        if attempts < 3 {
            return &TransientError{Err: errors.New("timeout")}
        }
        return nil
    }
    
    // Act
    err := interceptor.Execute(ctx, msg, handler)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, 3, attempts)
}
```

</td>
</tr>
</table>

## Integration Testing

### Testing with Real Message Infrastructure

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestClass]
public class MessagingIntegrationTests : IClassFixture<RabbitMqFixture>
{
    private readonly RabbitMqFixture _fixture;
    
    public MessagingIntegrationTests(RabbitMqFixture fixture)
    {
        _fixture = fixture;
    }
    
    [TestMethod]
    public async Task PublishAndConsume_Message_IsProcessed()
    {
        // Arrange
        var services = new ServiceCollection();
        services.AddMmateMessaging()
            .WithRabbitMqTransport(options =>
            {
                options.ConnectionString = _fixture.ConnectionString;
            });
        
        var received = new List<TestMessage>();
        services.AddMessageHandler<TestMessageHandler, TestMessage>();
        
        var provider = services.BuildServiceProvider();
        var publisher = provider.GetRequiredService<IMessagePublisher>();
        
        // Act
        await publisher.PublishAsync(new TestMessage { Id = "test-123" });
        
        // Wait for processing
        await Task.Delay(1000);
        
        // Assert
        Assert.AreEqual(1, received.Count);
        Assert.AreEqual("test-123", received[0].Id);
    }
}

public class RabbitMqFixture : IDisposable
{
    public string ConnectionString { get; private set; }
    
    public RabbitMqFixture()
    {
        // Start RabbitMQ container for testing
        ConnectionString = StartRabbitMqContainer();
    }
    
    public void Dispose()
    {
        // Clean up container
    }
}
```

</td>
<td>

```go
func TestMessagingIntegration_PublishAndConsume(t *testing.T) {
    // Arrange
    fixture := NewRabbitMQFixture(t)
    defer fixture.Cleanup()
    
    config := messaging.Config{
        RabbitMQ: rabbitmq.Config{
            URL: fixture.ConnectionString,
        },
    }
    
    dispatcher, err := messaging.NewDispatcher(config)
    require.NoError(t, err)
    defer dispatcher.Close()
    
    received := make(chan *TestMessage, 1)
    dispatcher.Handle("test.message", func(ctx context.Context, msg *TestMessage) error {
        received <- msg
        return nil
    })
    
    // Start consuming
    go dispatcher.Start(context.Background())
    time.Sleep(100 * time.Millisecond) // Wait for startup
    
    // Act
    err = dispatcher.Publish(context.Background(), &TestMessage{ID: "test-123"})
    require.NoError(t, err)
    
    // Assert
    select {
    case msg := <-received:
        assert.Equal(t, "test-123", msg.ID)
    case <-time.After(5 * time.Second):
        t.Fatal("Message not received within timeout")
    }
}

type RabbitMQFixture struct {
    ConnectionString string
    container        testcontainers.Container
}

func NewRabbitMQFixture(t *testing.T) *RabbitMQFixture {
    // Start RabbitMQ test container
    container, err := rabbitmq.RunContainer(
        context.Background(),
        testcontainers.WithImage("rabbitmq:3-management"),
    )
    require.NoError(t, err)
    
    connStr, err := container.ConnectionString(context.Background())
    require.NoError(t, err)
    
    return &RabbitMQFixture{
        ConnectionString: connStr,
        container:        container,
    }
}
```

</td>
</tr>
</table>

## Testing StageFlow Workflows

### Workflow Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestMethod]
public async Task OrderWorkflow_CompleteFlow_ProcessesSuccessfully()
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
    Assert.AreEqual(WorkflowStatus.Completed, result.Status);
}

[TestMethod]
public async Task OrderWorkflow_PaymentFails_TriggersCompensation()
{
    // Arrange
    var mockPayment = new Mock<IPaymentService>();
    mockPayment
        .Setup(p => p.ProcessPayment(It.IsAny<PaymentRequest>()))
        .ReturnsAsync(new PaymentResult { Success = false });
    
    var services = new ServiceCollection();
    services.AddStageFlow()
        .AddSingleton(mockPayment.Object);
    
    // Act
    var result = await workflow.ExecuteAsync(context);
    
    // Assert
    Assert.IsFalse(result.Success);
    // Verify compensation was called
    mockInventory.Verify(i => i.ReleaseReservation(It.IsAny<string>()), Times.Once);
}
```

</td>
<td>

```go
func TestOrderWorkflow_CompleteFlow_ProcessesSuccessfully(t *testing.T) {
    // Arrange
    workflow := NewOrderWorkflow()
    workflow.RegisterServices(
        &MockPaymentService{ShouldSucceed: true},
        &MockInventoryService{ShouldSucceed: true})
    
    context := &OrderContext{
        OrderID: "ORDER-123",
        Items: []OrderItem{
            {ProductID: "P1", Quantity: 2},
        },
    }
    
    // Act
    result, err := workflow.Execute(context.Background(), context)
    
    // Assert
    assert.NoError(t, err)
    assert.True(t, result.Success)
    assert.True(t, context.InventoryReserved)
    assert.NotEmpty(t, context.PaymentID)
    assert.Equal(t, stageflow.StatusCompleted, result.Status)
}

func TestOrderWorkflow_PaymentFails_TriggersCompensation(t *testing.T) {
    // Arrange
    mockInventory := &MockInventoryService{ShouldSucceed: true}
    workflow := NewOrderWorkflow()
    workflow.RegisterServices(
        &MockPaymentService{ShouldSucceed: false},
        mockInventory)
    
    // Act
    result, err := workflow.Execute(context.Background(), context)
    
    // Assert
    assert.Error(t, err)
    assert.False(t, result.Success)
    // Verify compensation was called
    assert.True(t, mockInventory.CompensationCalled)
}
```

</td>
</tr>
</table>

## Contract Testing

### Message Contract Validation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestClass]
public class MessageContractTests
{
    [TestMethod]
    public void OrderCreatedEvent_Serialization_MaintainsContract()
    {
        // Arrange
        var originalEvent = new OrderCreatedEvent
        {
            OrderId = "ORDER-123",
            CustomerId = "CUST-456",
            Items = new[] { new OrderItem { ProductId = "P1" } },
            Timestamp = DateTimeOffset.UtcNow
        };
        
        // Act - Serialize and deserialize
        var json = JsonSerializer.Serialize(originalEvent);
        var deserializedEvent = JsonSerializer.Deserialize<OrderCreatedEvent>(json);
        
        // Assert - Contract fields preserved
        Assert.AreEqual(originalEvent.OrderId, deserializedEvent.OrderId);
        Assert.AreEqual(originalEvent.CustomerId, deserializedEvent.CustomerId);
        Assert.AreEqual(originalEvent.Items.Length, deserializedEvent.Items.Length);
    }
    
    [DataTestMethod]
    [DataRow("OrderCreatedEvent", "1.0")]
    [DataRow("PaymentProcessedEvent", "2.1")]
    public void MessageSchema_Validation_PassesForSupportedVersions(
        string messageType, 
        string version)
    {
        // Arrange
        var schema = SchemaRegistry.GetSchema(messageType, version);
        var message = CreateTestMessage(messageType);
        
        // Act
        var validationResult = schema.Validate(message);
        
        // Assert
        Assert.IsTrue(validationResult.IsValid);
    }
}
```

</td>
<td>

```go
func TestOrderCreatedEvent_Serialization_MaintainsContract(t *testing.T) {
    // Arrange
    originalEvent := &OrderCreatedEvent{
        OrderID:    "ORDER-123",
        CustomerID: "CUST-456",
        Items: []OrderItem{
            {ProductID: "P1"},
        },
        Timestamp: time.Now(),
    }
    
    // Act - Serialize and deserialize
    data, err := json.Marshal(originalEvent)
    require.NoError(t, err)
    
    var deserializedEvent OrderCreatedEvent
    err = json.Unmarshal(data, &deserializedEvent)
    require.NoError(t, err)
    
    // Assert - Contract fields preserved
    assert.Equal(t, originalEvent.OrderID, deserializedEvent.OrderID)
    assert.Equal(t, originalEvent.CustomerID, deserializedEvent.CustomerID)
    assert.Len(t, deserializedEvent.Items, len(originalEvent.Items))
}

func TestMessageSchema_Validation_PassesForSupportedVersions(t *testing.T) {
    tests := []struct {
        messageType string
        version     string
    }{
        {"OrderCreatedEvent", "1.0"},
        {"PaymentProcessedEvent", "2.1"},
    }
    
    for _, tt := range tests {
        t.Run(fmt.Sprintf("%s_%s", tt.messageType, tt.version), func(t *testing.T) {
            // Arrange
            schema := schemaRegistry.GetSchema(tt.messageType, tt.version)
            message := createTestMessage(tt.messageType)
            
            // Act
            validationResult := schema.Validate(message)
            
            // Assert
            assert.True(t, validationResult.IsValid)
        })
    }
}
```

</td>
</tr>
</table>

## Load Testing

### Performance Testing Setup

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[TestClass]
public class LoadTests
{
    [TestMethod]
    public async Task MessageThroughput_HighVolume_MaintainsPerformance()
    {
        // Arrange
        const int messageCount = 10000;
        const int concurrency = 10;
        
        var services = new ServiceCollection();
        services.AddMmateMessaging()
            .WithRabbitMqTransport(options =>
            {
                options.MaxConcurrentMessages = concurrency;
                options.PrefetchCount = 100;
            });
        
        var provider = services.BuildServiceProvider();
        var publisher = provider.GetRequiredService<IMessagePublisher>();
        
        var stopwatch = Stopwatch.StartNew();
        var processed = 0;
        
        // Act
        var tasks = Enumerable.Range(0, messageCount)
            .Select(async i =>
            {
                await publisher.PublishAsync(new TestMessage { Id = i.ToString() });
                Interlocked.Increment(ref processed);
            });
        
        await Task.WhenAll(tasks);
        stopwatch.Stop();
        
        // Assert
        var throughput = messageCount / stopwatch.Elapsed.TotalSeconds;
        Assert.IsTrue(throughput > 1000, $"Throughput too low: {throughput:F2} msg/sec");
    }
}
```

</td>
<td>

```go
func TestMessageThroughput_HighVolume_MaintainsPerformance(t *testing.T) {
    // Arrange
    const messageCount = 10000
    const concurrency = 10
    
    config := messaging.Config{
        MaxConcurrentMessages: concurrency,
        RabbitMQ: rabbitmq.Config{
            PrefetchCount: 100,
        },
    }
    
    dispatcher, err := messaging.NewDispatcher(config)
    require.NoError(t, err)
    defer dispatcher.Close()
    
    var processed int64
    start := time.Now()
    
    // Act
    var wg sync.WaitGroup
    for i := 0; i < messageCount; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            err := dispatcher.Publish(context.Background(), 
                &TestMessage{ID: fmt.Sprintf("%d", id)})
            if err == nil {
                atomic.AddInt64(&processed, 1)
            }
        }(i)
    }
    
    wg.Wait()
    elapsed := time.Since(start)
    
    // Assert
    throughput := float64(atomic.LoadInt64(&processed)) / elapsed.Seconds()
    assert.Greater(t, throughput, 1000.0, 
        "Throughput too low: %.2f msg/sec", throughput)
}
```

</td>
</tr>
</table>

## Test Utilities

### Message Context Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public static class TestHelpers
{
    public static MessageContext CreateTestContext<T>(T message)
        where T : IMessage
    {
        var context = MessageContext.CreateForTesting();
        context.Message = message;
        context.SetHeader("MessageId", Guid.NewGuid().ToString());
        context.SetHeader("Timestamp", DateTimeOffset.UtcNow);
        return context;
    }
    
    public static async Task<T> PublishAndWaitAsync<T>(
        IMessagePublisher publisher,
        IMessage message,
        TimeSpan timeout = default)
        where T : IMessage
    {
        timeout = timeout == default ? TimeSpan.FromSeconds(5) : timeout;
        
        var received = new TaskCompletionSource<T>();
        
        // Set up temporary handler
        using var subscription = SetupTemporaryHandler<T>(received);
        
        await publisher.PublishAsync(message);
        
        return await received.Task.WaitAsync(timeout);
    }
}
```

</td>
<td>

```go
// Test utilities
func CreateTestContext(msg messaging.Message) context.Context {
    ctx := context.Background()
    ctx = context.WithValue(ctx, "messageId", uuid.New().String())
    ctx = context.WithValue(ctx, "timestamp", time.Now())
    return ctx
}

func PublishAndWait[T messaging.Message](
    t *testing.T,
    dispatcher messaging.Dispatcher,
    message messaging.Message,
    timeout time.Duration) T {
    
    if timeout == 0 {
        timeout = 5 * time.Second
    }
    
    received := make(chan T, 1)
    
    // Set up temporary handler
    dispatcher.Handle(message.Type(), func(ctx context.Context, msg T) error {
        received <- msg
        return nil
    })
    
    // Publish message
    err := dispatcher.Publish(context.Background(), message)
    require.NoError(t, err)
    
    // Wait for response
    select {
    case msg := <-received:
        return msg
    case <-time.After(timeout):
        t.Fatal("Message not received within timeout")
        var zero T
        return zero
    }
}
```

</td>
</tr>
</table>

## Best Practices

### Testing Checklist

1. **Unit Tests**
   - [ ] Test message handlers in isolation
   - [ ] Mock external dependencies
   - [ ] Test error scenarios
   - [ ] Validate message contracts

2. **Integration Tests**
   - [ ] Test with real message infrastructure
   - [ ] Use test containers for isolation
   - [ ] Test complete workflows
   - [ ] Verify message routing

3. **Performance Tests**
   - [ ] Test under expected load
   - [ ] Measure throughput and latency
   - [ ] Test with realistic message sizes
   - [ ] Monitor resource usage

4. **Contract Tests**
   - [ ] Validate message schemas
   - [ ] Test serialization/deserialization
   - [ ] Verify backward compatibility
   - [ ] Test cross-platform compatibility

### Testing Anti-Patterns

1. **Don't test infrastructure**: Focus on business logic, not framework behavior
2. **Avoid flaky tests**: Use deterministic timing and proper synchronization
3. **Don't ignore errors**: Test error paths as thoroughly as happy paths
4. **Avoid shared state**: Keep tests isolated and independent