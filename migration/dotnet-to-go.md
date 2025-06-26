# Migrating from .NET to Go

This guide helps .NET developers transition from Mmate .NET to Mmate Go implementation.

## Key Differences Overview

| Aspect | .NET | Go |
|--------|------|-----|
| Async Model | async/await | Goroutines + channels |
| Error Handling | Exceptions | Error returns |
| DI Container | Built-in ASP.NET Core | Manual wiring |
| Configuration | appsettings.json | Environment variables |
| Package Manager | NuGet | Go modules |

## Package Mapping

| .NET Package | Go Package | Notes |
|--------------|------------|-------|
| Mmate.Contracts | contracts | Same concepts |
| Mmate.Messaging | messaging | Core functionality |
| Mmate.StageFlow | stageflow | Workflow orchestration |
| Mmate.SyncAsyncBridge | bridge | Name change |
| Mmate.Interceptors | interceptors | Same patterns |
| Mmate.Messaging.Discovery | - | Removed (explicit registration) |

## Code Translation Examples

### 1. Message Definitions

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

### 2. Publishing Messages

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

**Go:**
```go
type OrderService struct {
    publisher messaging.Publisher
}

func NewOrderService(publisher messaging.Publisher) *OrderService {
    return &OrderService{publisher: publisher}
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

### 3. Message Handlers

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

**Go:**
```go
type OrderHandler struct {
    repository OrderRepository
    logger     *slog.Logger
}

func NewOrderHandler(repository OrderRepository, logger *slog.Logger) *OrderHandler {
    return &OrderHandler{
        repository: repository,
        logger:     logger,
    }
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

### 4. Dependency Injection

**.NET:**
```csharp
// Program.cs
builder.Services.AddMmate(options =>
{
    options.ConnectionString = "amqp://localhost";
});

builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<OrderHandler>();

// Handlers auto-discovered via reflection
```

**Go:**
```go
// main.go
func main() {
    // Manual wiring
    connManager := rabbitmq.NewConnectionManager("amqp://localhost")
    channelPool, _ := rabbitmq.NewChannelPool(connManager)
    
    publisher := messaging.NewMessagePublisher(
        rabbitmq.NewPublisher(channelPool))
    
    subscriber := messaging.NewMessageSubscriber(
        rabbitmq.NewSubscriber(channelPool))
    
    // Create dependencies
    repository := NewOrderRepository(db)
    handler := NewOrderHandler(repository, logger)
    
    // Register handler explicitly
    dispatcher := messaging.NewDispatcher()
    dispatcher.Register("CreateOrderCommand", handler)
}
```

### 5. Configuration

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

**Go (environment variables):**
```go
type Config struct {
    ConnectionString string        `env:"MMATE_CONNECTION_STRING" default:"amqp://localhost"`
    PrefetchCount    int          `env:"MMATE_PREFETCH_COUNT" default:"10"`
    MaxRetries       int          `env:"MMATE_MAX_RETRIES" default:"3"`
    RetryDelay       time.Duration `env:"MMATE_RETRY_DELAY" default:"1s"`
}

func LoadConfig() (*Config, error) {
    cfg := &Config{}
    if err := envconfig.Process("", cfg); err != nil {
        return nil, err
    }
    return cfg, nil
}
```

### 6. Error Handling

**.NET:**
```csharp
try
{
    await ProcessOrderAsync(order);
}
catch (ValidationException ex)
{
    _logger.LogWarning(ex, "Validation failed");
    throw; // Will trigger retry
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

**Go:**
```go
err := processOrder(ctx, order)
if err != nil {
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        logger.Warn("Validation failed", "error", err)
        return err // Will trigger retry
    }
    
    var dbErr *DatabaseError
    if errors.As(err, &dbErr) {
        logger.Error("Database error", "error", err)
        return NewTransientError("database unavailable", err)
    }
    
    logger.Error("Unexpected error", "error", err)
    return NewPermanentError("processing failed", err)
}
```

### 7. Async Patterns

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

## Testing Migration

### Unit Tests

**.NET:**
```csharp
[Fact]
public async Task HandleAsync_ValidCommand_SavesOrder()
{
    // Arrange
    var repository = new Mock<IOrderRepository>();
    var handler = new OrderHandler(repository.Object, NullLogger<OrderHandler>.Instance);
    var command = new CreateOrderCommand { CustomerId = "123" };
    
    // Act
    await handler.HandleAsync(command, new MessageContext());
    
    // Assert
    repository.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
}
```

**Go:**
```go
func TestHandle_ValidCommand_SavesOrder(t *testing.T) {
    // Arrange
    repository := &MockOrderRepository{
        SaveFunc: func(ctx context.Context, order *Order) error {
            return nil
        },
    }
    handler := NewOrderHandler(repository, slog.Default())
    command := &CreateOrderCommand{CustomerID: "123"}
    
    // Act
    err := handler.Handle(context.Background(), command)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, 1, repository.SaveCallCount)
}
```

## Common Pitfalls

1. **Nil Pointer Exceptions**
   - Go doesn't have constructors, initialize properly
   - Check for nil before use

2. **Error Handling**
   - Don't ignore errors
   - Wrap errors with context
   - Use error types for different handling

3. **Goroutine Leaks**
   - Always use context for cancellation
   - Ensure goroutines can exit

4. **Interface Satisfaction**
   - Go interfaces are implicit
   - Ensure types implement all methods

## Performance Considerations

- Go typically has lower memory footprint
- Goroutines are lighter than threads
- No JIT compilation overhead
- Manual memory management considerations

## Migration Checklist

- [ ] Map all message types
- [ ] Port message handlers
- [ ] Set up configuration
- [ ] Implement error handling
- [ ] Add logging and metrics
- [ ] Write tests
- [ ] Performance testing
- [ ] Update deployment scripts
- [ ] Document Go-specific patterns
- [ ] Train team on Go idioms