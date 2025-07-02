# Schema - Message Validation and Versioning

The Schema component provides message validation, versioning, and contract enforcement for both .NET and Go implementations.

## Overview

The Schema system enables:
- Runtime message validation
- Schema versioning for backward compatibility
- Contract-first development
- Auto-generation of documentation
- Type safety across services

## Core Concepts

### Schema Definition
Schemas define the structure and validation rules for messages, ensuring consistency across distributed systems.

### Validation
Messages are validated against their schemas before processing, preventing invalid data from entering the system.

### Versioning
Support for multiple schema versions allows gradual migration and backward compatibility.

## Basic Usage

### Defining Schemas

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[MessageSchema(Version = "1.0", 
    Owner = "OrderService")]
public class CreateOrderCommand : Command
{
    [Required]
    public Guid OrderId { get; set; }
    
    [Required]
    public Guid CustomerId { get; set; }
    
    [Required]
    [MinLength(1)]
    public List<OrderItem> Items { get; set; }
    
    [StringLength(500)]
    public string Notes { get; set; }
    
    public override ValidationResult Validate()
    {
        var errors = new List<string>();
        
        if (Items?.Any() != true)
            errors.Add("Order must have items");
            
        foreach (var item in Items ?? new())
        {
            if (item.Quantity <= 0)
                errors.Add($"Invalid quantity: {item.Quantity}");
        }
        
        return errors.Any() 
            ? ValidationResult.Failure(errors)
            : ValidationResult.Success();
    }
}

[MessageSchema(Version = "1.0")]
public class OrderItem
{
    [Required]
    public Guid ProductId { get; set; }
    
    [Range(1, 1000)]
    public int Quantity { get; set; }
    
    [Range(0.01, 100000)]
    public decimal UnitPrice { get; set; }
}
```

</td>
<td>

```go
// Schema defined via struct tags
type CreateOrderCommand struct {
    contracts.BaseCommand
    OrderID    string      `json:"orderId" validate:"required,uuid"`
    CustomerID string      `json:"customerId" validate:"required,uuid"`
    Items      []OrderItem `json:"items" validate:"required,min=1,dive"`
    Notes      string      `json:"notes" validate:"max=500"`
}

type OrderItem struct {
    ProductID string  `json:"productId" validate:"required,uuid"`
    Quantity  int     `json:"quantity" validate:"required,min=1,max=1000"`
    UnitPrice float64 `json:"unitPrice" validate:"required,min=0.01,max=100000"`
}

// Implement validation
func (c *CreateOrderCommand) Validate() error {
    // Use validator library
    v := validator.New()
    if err := v.Struct(c); err != nil {
        return err
    }
    
    // Custom validation
    for i, item := range c.Items {
        if item.Quantity <= 0 {
            return fmt.Errorf("item %d: invalid quantity %d", 
                i, item.Quantity)
        }
    }
    
    return nil
}

// Schema metadata via constants
const (
    CreateOrderCommandSchema = "v1.0"
    CreateOrderCommandOwner  = "OrderService"
)
```

</td>
</tr>
</table>

### Schema Registration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Automatic registration via attributes
services.AddMmateSchema(options =>
{
    options.ValidationMode = SchemaValidationMode.Strict;
    options.SchemaNamespace = "MyApp.Schemas";
    options.EnableAutoDiscovery = true;
});

// Manual registration
services.AddSingleton<ISchemaRegistry>(provider =>
{
    var registry = new SchemaRegistry();
    
    registry.Register<CreateOrderCommand>("1.0");
    registry.Register<CreateOrderCommand>("2.0");
    
    return registry;
});
```

</td>
<td>

```go
// Manual registration
func RegisterSchemas(registry schema.Registry) {
    registry.Register(
        "CreateOrderCommand",
        "1.0",
        schema.Schema{
            Type:  reflect.TypeOf(CreateOrderCommand{}),
            Owner: "OrderService",
            Validator: func(msg interface{}) error {
                cmd, ok := msg.(*CreateOrderCommand)
                if !ok {
                    return errors.New("invalid type")
                }
                return cmd.Validate()
            },
        })
}

// Using with validator
validator := schema.NewValidator(registry)
subscriber := messaging.NewMessageSubscriber(
    transport,
    messaging.WithSchemaValidator(validator))
```

</td>
</tr>
</table>

## Schema Versioning

### Versioning Strategy for mmate-go

Since mmate-go stores one schema per message type, developers should adopt these strategies:

1. **Message Type Versioning**: Include version in the message type name
2. **Additive Changes Only**: Only add optional fields to existing messages
3. **Explicit Version Handling**: Handle multiple versions explicitly in consumers
4. **Gradual Migration**: Support old versions during transition periods

### Version Evolution

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Version 1.0
[MessageSchema(Version = "1.0")]
public class OrderEvent : Event
{
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
}

// Version 2.0 - Added new field
[MessageSchema(Version = "2.0")]
public class OrderEventV2 : Event
{
    public string OrderId { get; set; }
    public decimal Amount { get; set; }
    public string Currency { get; set; } = "USD"; // Default for compatibility
    
    // Conversion from v1
    public static OrderEventV2 FromV1(OrderEvent v1)
    {
        return new OrderEventV2
        {
            OrderId = v1.OrderId,
            Amount = v1.Amount,
            Currency = "USD" // Default value
        };
    }
}

// Handler supporting multiple versions
public class OrderEventHandler : 
    IMessageHandler<OrderEvent>,
    IMessageHandler<OrderEventV2>
{
    public async Task HandleAsync(OrderEvent message, MessageContext context)
    {
        // Convert to latest version
        var v2 = OrderEventV2.FromV1(message);
        await ProcessOrderV2(v2);
    }
    
    public async Task HandleAsync(OrderEventV2 message, MessageContext context)
    {
        await ProcessOrderV2(message);
    }
}
```

</td>
<td>

```go
// Version 1.0
type OrderEvent struct {
    contracts.BaseEvent
    OrderID string  `json:"orderId"`
    Amount  float64 `json:"amount"`
}

// Version 2.0 - Added new field
type OrderEventV2 struct {
    contracts.BaseEvent
    OrderID  string  `json:"orderId"`
    Amount   float64 `json:"amount"`
    Currency string  `json:"currency,omitempty"`
}

// Conversion function
func (e *OrderEventV2) FromV1(v1 *OrderEvent) {
    e.OrderID = v1.OrderID
    e.Amount = v1.Amount
    e.Currency = "USD" // Default value
}

// Handler supporting multiple versions
type OrderEventHandler struct{}

func (h *OrderEventHandler) Handle(ctx context.Context, 
    msg contracts.Message) error {
    
    switch m := msg.(type) {
    case *OrderEvent:
        // Convert to v2
        v2 := &OrderEventV2{}
        v2.FromV1(m)
        return h.processOrderV2(ctx, v2)
        
    case *OrderEventV2:
        return h.processOrderV2(ctx, m)
        
    default:
        return fmt.Errorf("unsupported version: %T", msg)
    }
}
```

</td>
</tr>
</table>

## Validation Interceptor

### Automatic Validation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class SchemaValidationInterceptor : IMessageInterceptor
{
    private readonly ISchemaRegistry _registry;
    private readonly ISchemaValidator _validator;
    
    public async Task OnBeforeConsumeAsync(
        MessageContext context, 
        CancellationToken cancellationToken)
    {
        var messageType = context.Message.GetType();
        var schema = _registry.GetSchema(messageType);
        
        if (schema != null)
        {
            var result = _validator.Validate(
                context.Message, schema);
                
            if (!result.IsValid)
            {
                throw new SchemaValidationException(
                    $"Validation failed: {string.Join(", ", result.Errors)}");
            }
        }
    }
}

// Registration
services.AddMmateInterceptor<SchemaValidationInterceptor>();
```

</td>
<td>

```go
type SchemaValidationInterceptor struct {
    registry  schema.Registry
    validator schema.Validator
}

func (i *SchemaValidationInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Get schema for message type
    msgType := reflect.TypeOf(msg).String()
    schema, exists := i.registry.GetSchema(msgType)
    
    if exists {
        if err := i.validator.Validate(msg, schema); err != nil {
            return fmt.Errorf("schema validation failed: %w", err)
        }
    }
    
    return next(ctx, msg)
}

// Registration
pipeline := interceptors.NewPipeline(
    NewSchemaValidationInterceptor(registry),
    // other interceptors...
)
```

</td>
</tr>
</table>

### Practical Schema Versioning in Go

While mmate-go has type registration for serialization (`messaging.Register`), schema versioning requires additional implementation. Here's the recommended approach:

#### 1. Type Registration with Versioning

```go
// Register message types for serialization
messaging.Register("OrderEvent", func() contracts.Message {
    return &OrderEvent{}
})
messaging.Register("OrderEventV2", func() contracts.Message {
    return &OrderEventV2{}
})

// Create and configure schema validator
validator := schema.NewMessageValidator()

// Register schemas with version in the key
validator.RegisterSchema("OrderEvent", orderEventSchemaV1)
validator.RegisterSchema("OrderEventV2", orderEventSchemaV2)

// Or use versioned keys
validator.RegisterSchema("OrderEvent:v1.0", orderEventSchemaV1)
validator.RegisterSchema("OrderEvent:v2.0", orderEventSchemaV2)
```

#### 2. Version-Aware Message Handler
```go
type VersionAwareOrderHandler struct {
    validator *schema.MessageValidator
}

func (h *VersionAwareOrderHandler) Handle(ctx context.Context, msg contracts.Message) error {
    switch m := msg.(type) {
    case *OrderEvent:
        // Handle V1 - convert to V2 for processing
        v2 := h.migrateV1ToV2(m)
        return h.processOrder(ctx, v2)
    case *OrderEventV2:
        // Handle V2 directly
        return h.processOrder(ctx, m)
    default:
        return fmt.Errorf("unsupported message version: %T", msg)
    }
}

func (h *VersionAwareOrderHandler) migrateV1ToV2(v1 *OrderEvent) *OrderEventV2 {
    return &OrderEventV2{
        BaseEvent: v1.BaseEvent,
        OrderID:   v1.OrderID,
        Amount:    v1.Amount,
        Currency:  "USD", // Provide defaults for new fields
    }
}
```

#### 3. Schema Evolution Best Practices

**Safe Changes (Backward Compatible):**
- Add optional fields (with `omitempty` tag)
- Widen numeric types (int32 → int64)
- Change required field to optional

**Breaking Changes (Require New Version):**
- Remove fields
- Rename fields
- Change field types
- Make optional field required

#### 4. Deployment Strategy
```go
// Phase 1: Deploy consumers that handle both versions
subscriber.Subscribe("orders", &VersionAwareOrderHandler{})

// Phase 2: Deploy producers using new version
publisher.Publish(ctx, &OrderEventV2{...})

// Phase 3: After all producers upgraded, remove V1 support
// (keeping it for historical message replay if needed)
```

#### 5. Integrating Schema Versioning

Since schema versioning is not built into the Client interface, you need to implement it:

```go
// 1. Add schema version when publishing
err := publisher.Publish(ctx, &OrderEventV2{...},
    messaging.WithHeader("x-schema-version", "2.0"))

// 2. Create a schema-aware interceptor
type SchemaInterceptor struct {
    validator *schema.MessageValidator
}

func NewSchemaInterceptor() *SchemaInterceptor {
    validator := schema.NewMessageValidator()
    // Register your schemas
    validator.RegisterSchema("OrderEvent:v1.0", orderSchemaV1)
    validator.RegisterSchema("OrderEvent:v2.0", orderSchemaV2)
    return &SchemaInterceptor{validator: validator}
}

func (i *SchemaInterceptor) Intercept(ctx context.Context, msg contracts.Message, next interceptors.MessageHandler) error {
    // For outgoing messages: add schema version
    if publisher, ok := ctx.Value("publisher").(bool); ok && publisher {
        // Determine version from message type
        version := "1.0" // or detect from type name suffix
        if strings.HasSuffix(msg.GetType(), "V2") {
            version = "2.0"
        }
        // Add to publish options via context
        ctx = context.WithValue(ctx, "x-schema-version", version)
    }
    
    // For incoming messages: validate against schema
    return next.Handle(ctx, msg)
}

// 3. Use with Client
pipeline := interceptors.NewPipeline(
    NewSchemaInterceptor(),
    // other interceptors...
)

client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithInterceptors(pipeline))
```

#### 6. Best Practices for Schema Evolution

1. **Use Type Suffixes for Breaking Changes**:
   ```go
   messaging.Register("OrderEvent", func() contracts.Message { return &OrderEvent{} })
   messaging.Register("OrderEventV2", func() contracts.Message { return &OrderEventV2{} })
   ```

2. **Include Version in Headers**:
   ```go
   messaging.WithHeader("x-schema-version", "2.0")
   ```

3. **Handle Multiple Versions in Consumers**:
   - Check message type first (from envelope)
   - Then check schema version header
   - Have migration logic between versions

4. **Gradual Migration Strategy**:
   - Deploy consumers that handle both versions
   - Update producers to new version
   - Monitor and remove old version support later

## Advanced Features

### Custom Validators

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class BusinessRuleValidator : ISchemaValidator
{
    private readonly IBusinessRuleEngine _rules;
    
    public ValidationResult Validate(object message, Schema schema)
    {
        // Standard validation
        var result = base.Validate(message, schema);
        if (!result.IsValid) return result;
        
        // Business rule validation
        if (message is IBusinessValidatable validatable)
        {
            var businessErrors = _rules.Validate(validatable);
            if (businessErrors.Any())
            {
                return ValidationResult.Failure(businessErrors);
            }
        }
        
        return ValidationResult.Success();
    }
}
```

</td>
<td>

```go
type BusinessRuleValidator struct {
    rules BusinessRuleEngine
}

func (v *BusinessRuleValidator) Validate(
    msg interface{}, 
    schema schema.Schema) error {
    
    // Standard validation
    if err := schema.Validator(msg); err != nil {
        return err
    }
    
    // Business rule validation
    if validatable, ok := msg.(BusinessValidatable); ok {
        if err := v.rules.Validate(validatable); err != nil {
            return fmt.Errorf("business rule failed: %w", err)
        }
    }
    
    return nil
}
```

</td>
</tr>
</table>

### Schema Generation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Generate JSON Schema
var generator = new JsonSchemaGenerator();
var jsonSchema = generator.Generate<CreateOrderCommand>();

// Generate documentation
var docGen = new SchemaDocumentationGenerator();
var markdown = docGen.GenerateMarkdown(
    Assembly.GetExecutingAssembly());

// Generate TypeScript definitions
var tsGen = new TypeScriptGenerator();
var typescript = tsGen.Generate(
    Assembly.GetExecutingAssembly());
```

</td>
<td>

```go
// Generate JSON Schema
generator := schema.NewJSONSchemaGenerator()
jsonSchema, err := generator.Generate(
    reflect.TypeOf(CreateOrderCommand{}))

// JSON schemas can be used for:
// - API documentation
// - Client code generation
// - Contract testing
```

</td>
</tr>
</table>

## Configuration

### Schema Options

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class SchemaOptions
{
    public SchemaValidationMode ValidationMode { get; set; }
    public bool EnableAutoDiscovery { get; set; }
    public string SchemaNamespace { get; set; }
    public bool ThrowOnUnknownSchema { get; set; }
    public bool EnableSchemaEvolution { get; set; }
    public Dictionary<string, Version> MinimumVersions { get; set; }
}

public enum SchemaValidationMode
{
    None,      // No validation
    Loose,     // Log warnings only
    Strict     // Throw on validation failure
}
```

</td>
<td>

```go
type SchemaOptions struct {
    ValidationMode      ValidationMode
    EnableStrictMode    bool
    SchemaNamespace     string
    AllowUnknownSchemas bool
    EnableEvolution     bool
    MinimumVersions     map[string]string
}

type ValidationMode int

const (
    ValidationNone ValidationMode = iota
    ValidationLoose
    ValidationStrict
)
```

</td>
</tr>
</table>

## Best Practices

1. **Define Clear Contracts**
   - Use descriptive field names
   - Include validation attributes/tags
   - Document field purposes
   - Specify required vs optional fields

2. **Version Carefully**
   - Only add optional fields to existing versions
   - Create new versions for breaking changes
   - Provide migration paths between versions
   - Document version differences

3. **Validate Early**
   - Validate at message creation
   - Use interceptors for automatic validation
   - Provide clear error messages
   - Log validation failures

4. **Performance Considerations**
   - Cache compiled validators
   - Use lazy validation for large messages
   - Consider validation sampling in high-throughput scenarios
   - Profile validation overhead

5. **Testing**
   - Test all schema versions
   - Verify backward compatibility
   - Test validation edge cases
   - Include performance tests for validation

## Monitoring

### Schema Metrics

Both platforms should track:
- Validation success/failure rates
- Schema version distribution
- Validation performance metrics
- Unknown schema encounters

### Health Checks

Include schema registry health in system health checks:
- Registry availability
- Schema loading status
- Validator initialization
- Version compatibility status