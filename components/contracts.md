# Message Contracts

Message contracts define the structure and types of messages in Mmate. They ensure type safety and consistency across services.

## Overview

Mmate defines four fundamental message types:

| Type | Purpose | Direction | Response |
|------|---------|-----------|----------|
| **Command** | Request an action | One-to-one | Optional Reply |
| **Event** | Notify something happened | One-to-many | None |
| **Query** | Request information | One-to-one | Required Reply |
| **Reply** | Response to Command/Query | One-to-one | None |

## Message Type Hierarchy

```
Message (Interface)
├── Command
│   ├── BaseCommand
│   └── [Your Commands]
├── Event
│   ├── BaseEvent
│   └── [Your Events]
├── Query
│   ├── BaseQuery
│   └── [Your Queries]
└── Reply
    ├── BaseReply
    └── [Your Replies]
```

## Defining Messages

### Commands

Commands represent instructions to perform an action.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class CreateOrderCommand : Command
{
    public string CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public ShippingAddress Address { get; set; }
    public PaymentMethod Payment { get; set; }
    
    public CreateOrderCommand() 
        : base("CreateOrderCommand")
    {
        Items = new List<OrderItem>();
    }
}

// With validation
public class CreateOrderCommand : Command
{
    private string _customerId;
    
    public string CustomerId 
    { 
        get => _customerId;
        set => _customerId = value ?? 
            throw new ArgumentNullException();
    }
    
    public override ValidationResult Validate()
    {
        if (Items?.Any() != true)
            return ValidationResult.Error(
                "Order must have items");
                
        if (Items.Sum(i => i.Quantity) <= 0)
            return ValidationResult.Error(
                "Invalid quantities");
                
        return ValidationResult.Success();
    }
}
```

</td>
<td>

```go
type CreateOrderCommand struct {
    contracts.BaseCommand
    CustomerID string         `json:"customerId"`
    Items      []OrderItem    `json:"items"`
    Address    ShippingAddress `json:"address"`
    Payment    PaymentMethod  `json:"payment"`
}

// Constructor
func NewCreateOrderCommand(customerID string) *CreateOrderCommand {
    return &CreateOrderCommand{
        BaseCommand: contracts.NewBaseCommand(
            "CreateOrderCommand"),
        CustomerID:  customerID,
        Items:       make([]OrderItem, 0),
    }
}

// With validation
func (c *CreateOrderCommand) Validate() error {
    if c.CustomerID == "" {
        return errors.New("customer ID required")
    }
    
    if len(c.Items) == 0 {
        return errors.New("order must have items")
    }
    
    totalQty := 0
    for _, item := range c.Items {
        totalQty += item.Quantity
    }
    if totalQty <= 0 {
        return errors.New("invalid quantities")
    }
    
    return nil
}
```

</td>
</tr>
</table>

### Events

Events notify that something has happened.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderCreatedEvent : Event
{
    public string OrderId { get; set; }
    public string CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; }
    public OrderStatus Status { get; set; }
    
    // Event metadata
    public string AggregateId => OrderId;
    public int Version { get; set; }
    
    public OrderCreatedEvent() 
        : base("OrderCreatedEvent")
    {
        CreatedAt = DateTime.UtcNow;
        Status = OrderStatus.Pending;
    }
}

// Domain event with causation
public class OrderShippedEvent : Event
{
    public string OrderId { get; set; }
    public string TrackingNumber { get; set; }
    public string Carrier { get; set; }
    public DateTime ShippedAt { get; set; }
    
    // Causation tracking
    public string CausationId { get; set; }
    public string UserId { get; set; }
    
    public OrderShippedEvent(string causationId) 
        : base("OrderShippedEvent")
    {
        CausationId = causationId;
        ShippedAt = DateTime.UtcNow;
    }
}
```

</td>
<td>

```go
type OrderCreatedEvent struct {
    contracts.BaseEvent
    OrderID     string       `json:"orderId"`
    CustomerID  string       `json:"customerId"`
    TotalAmount float64      `json:"totalAmount"`
    CreatedAt   time.Time    `json:"createdAt"`
    Status      OrderStatus  `json:"status"`
    
    // Event metadata
    Version     int          `json:"version"`
}

// Constructor
func NewOrderCreatedEvent(orderID string) *OrderCreatedEvent {
    return &OrderCreatedEvent{
        BaseEvent:   contracts.NewBaseEvent(
            "OrderCreatedEvent", orderID),
        OrderID:     orderID,
        CreatedAt:   time.Now(),
        Status:      OrderStatusPending,
    }
}

// Domain event with causation
type OrderShippedEvent struct {
    contracts.BaseEvent
    OrderID        string    `json:"orderId"`
    TrackingNumber string    `json:"trackingNumber"`
    Carrier        string    `json:"carrier"`
    ShippedAt      time.Time `json:"shippedAt"`
    
    // Causation tracking
    CausationID    string    `json:"causationId"`
    UserID         string    `json:"userId"`
}

func NewOrderShippedEvent(
    orderID, causationID string) *OrderShippedEvent {
    return &OrderShippedEvent{
        BaseEvent:    contracts.NewBaseEvent(
            "OrderShippedEvent", orderID),
        OrderID:      orderID,
        CausationID:  causationID,
        ShippedAt:    time.Now(),
    }
}
```

</td>
</tr>
</table>

### Queries

Queries request information without side effects.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class GetOrderDetailsQuery : Query
{
    public string OrderId { get; set; }
    public bool IncludeItems { get; set; }
    public bool IncludeHistory { get; set; }
    
    public GetOrderDetailsQuery() 
        : base("GetOrderDetailsQuery")
    {
        IncludeItems = true;
    }
}

// Query with pagination
public class SearchOrdersQuery : Query
{
    public string CustomerId { get; set; }
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public OrderStatus? Status { get; set; }
    
    // Pagination
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
    public string SortBy { get; set; } = "CreatedAt";
    public bool Descending { get; set; } = true;
}
```

</td>
<td>

```go
type GetOrderDetailsQuery struct {
    contracts.BaseQuery
    OrderID        string `json:"orderId"`
    IncludeItems   bool   `json:"includeItems"`
    IncludeHistory bool   `json:"includeHistory"`
}

func NewGetOrderDetailsQuery(
    orderID string) *GetOrderDetailsQuery {
    return &GetOrderDetailsQuery{
        BaseQuery:     contracts.NewBaseQuery(
            "GetOrderDetailsQuery"),
        OrderID:       orderID,
        IncludeItems:  true,
    }
}

// Query with pagination
type SearchOrdersQuery struct {
    contracts.BaseQuery
    CustomerID  string       `json:"customerId"`
    StartDate   *time.Time   `json:"startDate,omitempty"`
    EndDate     *time.Time   `json:"endDate,omitempty"`
    Status      *OrderStatus `json:"status,omitempty"`
    
    // Pagination
    Page        int          `json:"page"`
    PageSize    int          `json:"pageSize"`
    SortBy      string       `json:"sortBy"`
    Descending  bool         `json:"descending"`
}

func NewSearchOrdersQuery() *SearchOrdersQuery {
    return &SearchOrdersQuery{
        BaseQuery:  contracts.NewBaseQuery(
            "SearchOrdersQuery"),
        Page:       1,
        PageSize:   20,
        SortBy:     "createdAt",
        Descending: true,
    }
}
```

</td>
</tr>
</table>

### Replies

Replies respond to commands or queries.

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Success reply
public class OrderDetailsReply : Reply
{
    public OrderDto Order { get; set; }
    public List<OrderItemDto> Items { get; set; }
    public List<OrderEventDto> History { get; set; }
    
    public OrderDetailsReply() 
        : base(success: true)
    {
        Items = new List<OrderItemDto>();
        History = new List<OrderEventDto>();
    }
}

// Error reply
public class OrderCommandReply : Reply
{
    public string OrderId { get; set; }
    public string Message { get; set; }
    public Dictionary<string, string> Errors { get; set; }
    
    public static OrderCommandReply Success(
        string orderId)
    {
        return new OrderCommandReply
        {
            Success = true,
            OrderId = orderId,
            Message = "Order processed successfully"
        };
    }
    
    public static OrderCommandReply Failure(
        string message, 
        Dictionary<string, string> errors = null)
    {
        return new OrderCommandReply
        {
            Success = false,
            Message = message,
            Errors = errors ?? new()
        };
    }
}
```

</td>
<td>

```go
// Success reply
type OrderDetailsReply struct {
    contracts.BaseReply
    Order   *OrderDto        `json:"order"`
    Items   []OrderItemDto   `json:"items"`
    History []OrderEventDto  `json:"history"`
}

func NewOrderDetailsReply(
    requestID, correlationID string) *OrderDetailsReply {
    return &OrderDetailsReply{
        BaseReply: contracts.NewBaseReply(
            requestID, correlationID),
        Items:     make([]OrderItemDto, 0),
        History:   make([]OrderEventDto, 0),
    }
}

// Error reply
type OrderCommandReply struct {
    contracts.BaseReply
    OrderID string            `json:"orderId,omitempty"`
    Message string            `json:"message"`
    Errors  map[string]string `json:"errors,omitempty"`
}

func SuccessReply(
    requestID, correlationID, orderID string) *OrderCommandReply {
    reply := &OrderCommandReply{
        BaseReply: contracts.NewBaseReply(
            requestID, correlationID),
        OrderID:   orderID,
        Message:   "Order processed successfully",
    }
    reply.Success = true
    return reply
}

func ErrorReply(
    requestID, correlationID, message string,
    errors map[string]string) *OrderCommandReply {
    reply := &OrderCommandReply{
        BaseReply: contracts.NewBaseReply(
            requestID, correlationID),
        Message:   message,
        Errors:    errors,
    }
    reply.Success = false
    return reply
}
```

</td>
</tr>
</table>

## Message Metadata

All messages include standard metadata:

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Unique message identifier |
| `type` | string | Message type name |
| `timestamp` | ISO 8601 | When message was created |
| `correlationId` | string | For tracing related messages |

### Custom Headers

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Add custom headers
message.Headers["userId"] = currentUser.Id;
message.Headers["tenantId"] = tenantContext.Id;
message.Headers["source"] = "web-api";

// Read headers in handler
public async Task HandleAsync(
    OrderCommand cmd, 
    MessageContext context)
{
    var userId = context.Headers["userId"];
    var tenantId = context.Headers["tenantId"];
}
```

</td>
<td>

```go
// Add custom headers
msg.SetHeader("userId", currentUser.ID)
msg.SetHeader("tenantId", tenantContext.ID)
msg.SetHeader("source", "web-api")

// Read headers in handler
func (h *Handler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    userId := msg.GetHeader("userId")
    tenantId := msg.GetHeader("tenantId")
    
    return nil
}
```

</td>
</tr>
</table>

## Serialization

Messages are serialized to JSON with the following envelope:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "CreateOrderCommand",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "req-123",
  "headers": {
    "userId": "user-456",
    "source": "web-api"
  },
  "payload": {
    "customerId": "CUST-789",
    "items": [
      {
        "productId": "PROD-123",
        "quantity": 2,
        "price": 29.99
      }
    ]
  }
}
```

## Versioning

### Strategy 1: Type Name Versioning

Include version in message type name:

```csharp
// .NET
public class CreateOrderCommandV2 : Command { }

// Go
type CreateOrderCommandV2 struct { }
```

### Strategy 2: Schema Evolution

Add optional fields for backward compatibility:

```csharp
// .NET
public class OrderCreatedEvent : Event
{
    // Original fields
    public string OrderId { get; set; }
    
    // Added in v2 (optional)
    public string ExternalOrderId { get; set; }
    
    // Added in v3 (with default)
    public OrderPriority Priority { get; set; } 
        = OrderPriority.Normal;
}
```

### Strategy 3: Message Transformation

Transform old messages to new format:

```go
// Go
func TransformOrderEvent(old *OrderEventV1) *OrderEventV2 {
    return &OrderEventV2{
        // Map old fields
        OrderID: old.OrderID,
        // Set defaults for new fields
        Priority: PriorityNormal,
    }
}
```

## Best Practices

### Design Guidelines

1. **Immutability** - Messages should be immutable after creation
2. **Self-Contained** - Include all data needed to process
3. **Backward Compatible** - Don't break existing consumers
4. **Meaningful Names** - Use clear, descriptive type names

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Command | `VerbNounCommand` | `CreateOrderCommand` |
| Event | `NounVerbedEvent` | `OrderCreatedEvent` |
| Query | `GetNounQuery` | `GetOrderDetailsQuery` |
| Reply | `NounReply` | `OrderDetailsReply` |

### Validation

Always validate messages:

```csharp
// .NET
public override ValidationResult Validate()
{
    var errors = new List<string>();
    
    if (string.IsNullOrEmpty(CustomerId))
        errors.Add("Customer ID required");
        
    if (!Items.Any())
        errors.Add("Order must have items");
        
    return errors.Any() 
        ? ValidationResult.Error(errors)
        : ValidationResult.Success();
}
```

```go
// Go
func (c *CreateOrderCommand) Validate() error {
    var errs []string
    
    if c.CustomerID == "" {
        errs = append(errs, "customer ID required")
    }
    
    if len(c.Items) == 0 {
        errs = append(errs, "order must have items")
    }
    
    if len(errs) > 0 {
        return fmt.Errorf("validation failed: %s",
            strings.Join(errs, "; "))
    }
    
    return nil
}
```

## Testing Contracts

### Unit Tests

```csharp
// .NET
[Test]
public void CreateOrderCommand_Validates_Items()
{
    var command = new CreateOrderCommand
    {
        CustomerId = "CUST-123",
        Items = new List<OrderItem>()
    };
    
    var result = command.Validate();
    
    Assert.IsFalse(result.IsValid);
    Assert.Contains("Order must have items", 
        result.Errors);
}
```

```go
// Go
func TestCreateOrderCommand_Validate(t *testing.T) {
    cmd := &CreateOrderCommand{
        CustomerID: "CUST-123",
        Items:      []OrderItem{},
    }
    
    err := cmd.Validate()
    
    assert.Error(t, err)
    assert.Contains(t, err.Error(), 
        "order must have items")
}
```

### Serialization Tests

Always test serialization round-trips:

```go
func TestOrderEvent_Serialization(t *testing.T) {
    original := NewOrderCreatedEvent("ORDER-123")
    original.CustomerID = "CUST-456"
    original.TotalAmount = 99.99
    
    // Serialize
    data, err := json.Marshal(original)
    assert.NoError(t, err)
    
    // Deserialize
    var decoded OrderCreatedEvent
    err = json.Unmarshal(data, &decoded)
    assert.NoError(t, err)
    
    // Verify
    assert.Equal(t, original.OrderID, decoded.OrderID)
    assert.Equal(t, original.CustomerID, decoded.CustomerID)
    assert.Equal(t, original.TotalAmount, decoded.TotalAmount)
}
```

## Endpoint Contract Discovery

Mmate supports automatic endpoint discovery, allowing services to dynamically discover and communicate with each other without hardcoded endpoints. This feature is inspired by MATS3.io's decentralized approach.

### How It Works

Each service owns and publishes its own endpoint contracts:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Register an endpoint
await client.RegisterEndpointAsync(
    new EndpointContract
    {
        EndpointId = "order.validate",
        Version = "1.0.0",
        Queue = "order-service.validate",
        InputType = "ValidateOrderCommand",
        OutputType = "ValidationResult",
        Description = "Validates order details"
    });

// Discover endpoints
var contract = await client.DiscoverEndpointAsync(
    "order.validate");

// Discover by pattern
var contracts = await client.DiscoverEndpointsAsync(
    "order.*", version: "1.0.0");
```

</td>
<td>

```go
// Register an endpoint
client.RegisterEndpoint(ctx, &contracts.EndpointContract{
    EndpointID:  "order.validate",
    Version:     "1.0.0",
    Queue:       "order-service.validate",
    InputType:   "ValidateOrderCommand",
    OutputType:  "ValidationResult",
    Description: "Validates order details",
})

// Discover endpoints
contract, err := client.DiscoverEndpoint(ctx, 
    "order.validate")

// Discover by pattern
contracts, err := client.DiscoverEndpoints(ctx, 
    "order.*", "1.0.0")
```

</td>
</tr>
</table>

### Endpoint Contract Structure

```json
{
  "endpointId": "payment.process",
  "version": "2.0.0",
  "serviceName": "payment-service",
  "queue": "payment-service.process.v2",
  "inputType": "ProcessPaymentCommand",
  "outputType": "PaymentResult",
  "description": "Process payment with 3D secure",
  "deprecated": false,
  "metadata": {
    "sla": "500ms",
    "authentication": "required"
  },
  "inputSchema": { /* JSON Schema */ },
  "outputSchema": { /* JSON Schema */ }
}
```

### Discovery Protocol

1. **Registration**: Services register their endpoints on startup
2. **Announcement**: Periodic broadcasts of available endpoints
3. **Discovery**: Query for endpoints by ID or pattern
4. **Caching**: Local caching of discovered contracts

### Benefits

- **Decentralized**: No central registry required
- **Dynamic**: Services can be added/removed without configuration changes
- **Type-Safe**: Contracts include schemas for validation
- **Versioned**: Support for multiple versions simultaneously
- **Resilient**: Works even if discovery service is temporarily unavailable

### Using Discovered Endpoints

```go
// Discover and use an endpoint
contract, err := client.DiscoverEndpoint(ctx, "inventory.check")
if err != nil {
    return err
}

// Send message to discovered endpoint
err = client.PublishCommand(ctx, 
    CheckInventoryCommand{SKU: "ABC123"},
    mmate.WithDirectQueue(contract.Queue))
```

### Best Practices

1. **Register on Startup**: Register all endpoints when service starts
2. **Use Patterns**: Discover groups of related endpoints
3. **Handle Missing**: Gracefully handle missing endpoints
4. **Cache Results**: Cache discovered contracts locally
5. **Version Carefully**: Use semantic versioning for contracts

## Next Steps

- Learn about [Message Handlers](../messaging/README.md#handlers)
- Explore [Message Patterns](../../patterns/README.md)
- Review [Platform Examples](../../README.md#examples)