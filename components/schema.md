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

// Generate OpenAPI spec
openAPIGen := schema.NewOpenAPIGenerator()
spec, err := openAPIGen.Generate(registry)

// Generate protobuf definitions
protoGen := schema.NewProtobufGenerator()
proto, err := protoGen.Generate(registry)
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