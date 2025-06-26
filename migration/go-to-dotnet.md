# Migrating from Go to .NET

This guide helps Go developers transition from Mmate Go to the .NET implementation.

## Key Differences Overview

| Aspect | Go | .NET |
|--------|-----|------|
| Async Model | Goroutines + channels | async/await |
| Error Handling | Error returns | Exceptions |
| DI Container | Manual wiring | Built-in ASP.NET Core |
| Configuration | Environment variables | appsettings.json |
| Package Manager | Go modules | NuGet |

## Package Mapping

| Go Package | .NET Package | Notes |
|------------|--------------|-------|
| contracts | Mmate.Contracts | Same concepts |
| messaging | Mmate.Messaging | Core functionality |
| stageflow | Mmate.StageFlow | Workflow orchestration |
| bridge | Mmate.SyncAsyncBridge | Name change |
| interceptors | Mmate.Interceptors | Same patterns |
| - | Mmate.Messaging.Discovery | Added (auto-discovery) |

## Code Translation Examples

### 1. Message Definitions

**Go:**
```go
type CreateOrderCommand struct {
    contracts.BaseCommand
    CustomerID  string      `json:"customerId"`
    Items       []OrderItem `json:"items"`
    TotalAmount float64     `json:"totalAmount"`
}

func NewCreateOrderCommand() *CreateOrderCommand {
    return &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand("CreateOrderCommand"),
        Items:       make([]OrderItem, 0),
    }
}
```

**.NET:**
```csharp
public class CreateOrderCommand : Command
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
    
    public CreateOrderCommand() : base("CreateOrderCommand")
    {
        Items = new List<OrderItem>();
    }
}
```

### 2. Publishing Messages

**Go:**
```go
type OrderService struct {
    publisher messaging.Publisher
}

func (s *OrderService) CreateOrder(ctx context.Context, dto CreateOrderDto) error {
    command := &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand("CreateOrderCommand"),
        CustomerID:  dto.CustomerID,
        Items:       dto.Items,
    }
    
    return s.publisher.PublishCommand(ctx, command)
}
```

**.NET:**
```csharp
public class OrderService
{
    private readonly IMessagePublisher _publisher;
    
    public OrderService(IMessagePublisher publisher)
    {
        _publisher = publisher;
    }
    
    public async Task CreateOrderAsync(CreateOrderDto dto)
    {
        var command = new CreateOrderCommand
        {
            CustomerId = dto.CustomerId,
            Items = dto.Items
        };
        
        await _publisher.PublishCommandAsync(command);
    }
}
```

### 3. Message Handlers

**Go:**
```go
type OrderHandler struct {
    repository OrderRepository
    logger     *slog.Logger
}

func (h *OrderHandler) Handle(ctx context.Context, msg contracts.Message) error {
    command, ok := msg.(*CreateOrderCommand)
    if !ok {
        return fmt.Errorf("unexpected message type: %T", msg)
    }
    
    h.logger.Info("Processing order", "customerId", command.CustomerID)
    
    order := &Order{
        CustomerID: command.CustomerID,
        Items:      command.Items,
        CreatedAt:  time.Now(),
    }
    
    return h.repository.Save(ctx, order)
}
```

**.NET:**
```csharp
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderHandler> _logger;
    
    public OrderHandler(
        IOrderRepository repository,
        ILogger<OrderHandler> logger)
    {
        _repository = repository;
        _logger = logger;
    }
    
    public async Task HandleAsync(
        CreateOrderCommand command,
        MessageContext context)
    {
        _logger.LogInformation("Processing order for {CustomerId}", 
            command.CustomerId);
        
        var order = new Order
        {
            CustomerId = command.CustomerId,
            Items = command.Items,
            CreatedAt = DateTime.UtcNow
        };
        
        await _repository.SaveAsync(order);
    }
}
```

### 4. Dependency Injection

**Go (Manual):**
```go
func main() {
    // Manual wiring
    connManager := rabbitmq.NewConnectionManager("amqp://localhost")
    channelPool, _ := rabbitmq.NewChannelPool(connManager)
    
    publisher := messaging.NewMessagePublisher(
        rabbitmq.NewPublisher(channelPool))
    
    repository := NewOrderRepository(db)
    handler := NewOrderHandler(repository, logger)
    
    dispatcher := messaging.NewDispatcher()
    dispatcher.Register("CreateOrderCommand", handler)
}
```

**.NET (Automatic):**
```csharp
// Program.cs
builder.Services.AddMmate(options =>
{
    options.ConnectionString = "amqp://localhost";
});

builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<OrderHandler>();

// Handlers discovered automatically via reflection or explicit registration
builder.Services.AddMmateHandlers(typeof(OrderHandler).Assembly);
```

### 5. Configuration

**Go (Environment):**
```go
type Config struct {
    ConnectionString string        `env:"MMATE_CONNECTION_STRING"`
    PrefetchCount    int          `env:"MMATE_PREFETCH_COUNT"`
    MaxRetries       int          `env:"MMATE_MAX_RETRIES"`
    RetryDelay       time.Duration `env:"MMATE_RETRY_DELAY"`
}

cfg := &Config{}
envconfig.Process("", cfg)
```

**.NET (appsettings.json):**
```json
{
  "Mmate": {
    "ConnectionString": "amqp://localhost",
    "PrefetchCount": 10,
    "RetryPolicy": {
      "MaxAttempts": 3,
      "InitialDelay": "00:00:01"
    }
  }
}
```

```csharp
builder.Services.Configure<MmateOptions>(
    builder.Configuration.GetSection("Mmate"));
```

### 6. Error Handling

**Go:**
```go
err := processOrder(ctx, order)
if err != nil {
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        logger.Warn("Validation failed", "error", err)
        return err
    }
    
    var dbErr *DatabaseError
    if errors.As(err, &dbErr) {
        logger.Error("Database error", "error", err)
        return NewTransientError("database unavailable", err)
    }
    
    return fmt.Errorf("processing failed: %w", err)
}
```

**.NET:**
```csharp
try
{
    await ProcessOrderAsync(order);
}
catch (ValidationException ex)
{
    _logger.LogWarning(ex, "Validation failed");
    throw;
}
catch (DatabaseException ex)
{
    _logger.LogError(ex, "Database error");
    throw new TransientException("Database unavailable", ex);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Unexpected error");
    throw new PermanentException("Processing failed", ex);
}
```

### 7. Async Patterns

**Go:**
```go
func (s *Service) GetOrder(ctx context.Context, orderID string) (*OrderDto, error) {
    query := &GetOrderQuery{
        BaseQuery: contracts.NewBaseQuery("GetOrderQuery"),
        OrderID:   orderID,
    }
    
    reply, err := s.bridge.SendAndWait(ctx, query, 
        "qry.orders.get", 30*time.Second)
    if err != nil {
        return nil, err
    }
    
    orderReply := reply.(*OrderReply)
    if !orderReply.Success {
        return nil, &OrderNotFoundError{OrderID: orderID}
    }
    
    return orderReply.Order, nil
}
```

**.NET:**
```csharp
public async Task<OrderDto> GetOrderAsync(string orderId)
{
    var query = new GetOrderQuery { OrderId = orderId };
    
    var reply = await _bridge.SendAndWaitAsync<OrderReply>(
        query, 
        TimeSpan.FromSeconds(30));
    
    if (!reply.Success)
    {
        throw new OrderNotFoundException(orderId);
    }
    
    return reply.Order;
}
```

## .NET-Specific Features

### 1. Attribute-Based Configuration

```csharp
[MessageHandler(Queue = "orders", PrefetchCount = 10)]
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    // Handler implementation
}

[RetryPolicy(MaxAttempts = 3, BackoffMultiplier = 2)]
public async Task HandleAsync(CreateOrderCommand command)
{
    // Method with retry
}
```

### 2. Middleware Pipeline

```csharp
app.UseMmate(pipeline =>
{
    pipeline.UseLogging();
    pipeline.UseMetrics();
    pipeline.UseValidation();
    pipeline.UseAuthorization();
});
```

### 3. Health Checks Integration

```csharp
builder.Services.AddHealthChecks()
    .AddRabbitMQ()
    .AddCheck<CustomHealthCheck>("custom");

app.MapHealthChecks("/health");
```

## Testing Migration

### Unit Tests

**Go:**
```go
func TestHandle_ValidCommand_SavesOrder(t *testing.T) {
    repository := &MockOrderRepository{
        SaveFunc: func(ctx context.Context, order *Order) error {
            return nil
        },
    }
    handler := NewOrderHandler(repository, slog.Default())
    
    err := handler.Handle(context.Background(), 
        &CreateOrderCommand{CustomerID: "123"})
    
    assert.NoError(t, err)
    assert.Equal(t, 1, repository.SaveCallCount)
}
```

**.NET:**
```csharp
[Fact]
public async Task HandleAsync_ValidCommand_SavesOrder()
{
    // Arrange
    var repository = new Mock<IOrderRepository>();
    var handler = new OrderHandler(repository.Object, 
        NullLogger<OrderHandler>.Instance);
    var command = new CreateOrderCommand { CustomerId = "123" };
    
    // Act
    await handler.HandleAsync(command, new MessageContext());
    
    // Assert
    repository.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
}
```

## Benefits of .NET

1. **Rich Ecosystem**
   - Extensive library support
   - Mature tooling
   - IDE integration

2. **Language Features**
   - LINQ for data queries
   - Pattern matching
   - Nullable reference types

3. **Framework Integration**
   - ASP.NET Core integration
   - Built-in DI container
   - Configuration system

4. **Developer Experience**
   - IntelliSense
   - Refactoring tools
   - Debugging support

## Migration Checklist

- [ ] Set up .NET development environment
- [ ] Create new .NET project structure
- [ ] Port message contracts
- [ ] Implement handlers with DI
- [ ] Configure appsettings.json
- [ ] Set up health checks
- [ ] Migrate tests to xUnit/NUnit
- [ ] Configure CI/CD for .NET
- [ ] Update deployment scripts
- [ ] Train team on C# and .NET patternsquick-start.md