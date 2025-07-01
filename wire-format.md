# Wire Format Specification

This document defines the wire format used by Mmate for cross-language compatibility between .NET and Go implementations.

## Message Envelope

All messages are wrapped in a JSON envelope with the following structure:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "CreateOrderCommand",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "req-123",
  "replyTo": "rpl.550e8400-e29b-41d4-a716-446655440000",
  "headers": {
    "userId": "user-456",
    "tenantId": "tenant-789",
    "source": "web-api"
  },
  "payload": {
    // Message-specific content
  }
}
```

## Envelope Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (UUID) | Yes | Unique message identifier (UUID v4 with hyphens) |
| `type` | string | Yes | Message type name for routing |
| `timestamp` | string (ISO 8601) | Yes | When message was created |
| `correlationId` | string | No | For tracing related messages |
| `replyTo` | string | No | Queue name for replies (commands/queries) |
| `headers` | object | No | Custom key-value pairs |
| `payload` | object | Yes | Message-specific data |

## Data Types

### Timestamps
- Format: ISO 8601 with timezone
- Example: `2024-01-15T10:30:00Z`
- Always UTC

### UUIDs
- Format: UUID v4 with hyphens
- Example: `550e8400-e29b-41d4-a716-446655440000`
- Lowercase

### Enums
- Serialized as strings
- Use exact casing from source language

## Message Types

### Command
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "CreateOrderCommand",
  "timestamp": "2024-01-15T10:30:00Z",
  "correlationId": "req-123",
  "replyTo": "rpl.550e8400-e29b-41d4-a716-446655440000",
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

### Event
```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "type": "OrderCreatedEvent",
  "timestamp": "2024-01-15T10:31:00Z",
  "correlationId": "req-123",
  "headers": {
    "aggregateId": "ORDER-456",
    "version": "1"
  },
  "payload": {
    "orderId": "ORDER-456",
    "customerId": "CUST-789",
    "totalAmount": 59.98,
    "createdAt": "2024-01-15T10:31:00Z"
  }
}
```

### Query
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440002",
  "type": "GetOrderDetailsQuery",
  "timestamp": "2024-01-15T10:32:00Z",
  "correlationId": "req-124",
  "replyTo": "rpl.770e8400-e29b-41d4-a716-446655440002",
  "payload": {
    "orderId": "ORDER-456",
    "includeItems": true
  }
}
```

### Reply
```json
{
  "id": "880e8400-e29b-41d4-a716-446655440003",
  "type": "OrderDetailsReply",
  "timestamp": "2024-01-15T10:32:05Z",
  "correlationId": "req-124",
  "headers": {
    "requestId": "770e8400-e29b-41d4-a716-446655440002"
  },
  "payload": {
    "success": true,
    "order": {
      "orderId": "ORDER-456",
      "customerId": "CUST-789",
      "status": "Processing",
      "items": [
        {
          "productId": "PROD-123",
          "quantity": 2,
          "price": 29.99
        }
      ]
    }
  }
}
```

## AMQP Properties Mapping

| Envelope Field | AMQP Property | Notes |
|----------------|---------------|-------|
| `id` | MessageId | Direct mapping |
| `correlationId` | CorrelationId | Direct mapping |
| `timestamp` | Timestamp | Unix timestamp in AMQP |
| `type` | Type | Direct mapping |
| `replyTo` | ReplyTo | Direct mapping |
| `headers.*` | Headers | Flattened in AMQP headers |

## Queue Naming Conventions

### Command Queues
Pattern: `cmd.{service}.{command}`
- Example: `cmd.orders.create`
- Example: `cmd.inventory.reserve`

### Event Queues
Pattern: `evt.{aggregate}.{event}`
- Example: `evt.order.created`
- Example: `evt.payment.processed`

### Query Queues
Pattern: `qry.{service}.{query}`
- Example: `qry.catalog.products`
- Example: `qry.pricing.calculate`

### Reply Queues
Pattern: `rpl.{correlationId}`
- Example: `rpl.550e8400-e29b-41d4-a716-446655440000`
- Temporary queues, auto-delete

### Dead Letter Queues
Pattern: `dlq.{originalQueue}`
- Example: `dlq.cmd.orders.create`
- Example: `dlq.evt.order.created`

### StageFlow Queues
Pattern: `stageflow.{workflowId}.stage{index}`
- Example: `stageflow.order-processing.stage1`
- Example: `stageflow.payment-flow.stage3`

## Exchange Configuration

### Topic Exchange
- Name: `mmate.topic`
- Type: `topic`
- Durable: `true`
- Used for event routing

### Direct Exchange
- Name: `mmate.direct`
- Type: `direct`
- Durable: `true`
- Used for command/query routing

### Headers Exchange
- Name: `mmate.headers`
- Type: `headers`
- Durable: `true`
- Used for complex routing

### Contracts Exchange
- Name: `mmate.contracts`
- Type: `topic`
- Durable: `true`
- Used for service contract discovery
- Routing pattern: `contract.announce`

## Routing Patterns

### Event Routing
- Routing key: `{aggregate}.{event}.{context}`
- Example: `order.created.us-west`
- Binding patterns:
  - `order.*.*` - All order events
  - `*.created.*` - All created events
  - `*.*.us-west` - All US West events

### Command Routing
- Routing key: Queue name
- Example: `cmd.orders.create`
- Direct routing to specific queue

## Error Handling

### Dead Letter Headers
Messages sent to DLQ include:
- `x-death`: Array of death records
- `x-death-reason`: Rejection reason
- `x-death-queue`: Original queue
- `x-death-time`: Failure timestamp

### Error Reply Format
```json
{
  "id": "990e8400-e29b-41d4-a716-446655440004",
  "type": "ErrorReply",
  "timestamp": "2024-01-15T10:33:00Z",
  "correlationId": "req-125",
  "payload": {
    "success": false,
    "errorCode": "VALIDATION_ERROR",
    "errorMessage": "Invalid order items",
    "errorDetails": {
      "field": "items[0].quantity",
      "message": "Quantity must be greater than 0"
    }
  }
}
```

## Serialization Rules

### JSON Serialization
1. Use camelCase for property names
2. Omit null values
3. Format dates as ISO 8601
4. Serialize enums as strings
5. Use UTF-8 encoding

### Type Mappings

| .NET Type | Go Type | JSON Type | Example |
|-----------|---------|-----------|---------|
| `string` | `string` | string | `"hello"` |
| `int`/`long` | `int`/`int64` | number | `42` |
| `decimal` | `float64` | number | `29.99` |
| `DateTime` | `time.Time` | string | `"2024-01-15T10:30:00Z"` |
| `Guid` | `string` | string | `"550e8400..."` |
| `bool` | `bool` | boolean | `true` |
| `List<T>` | `[]T` | array | `[1, 2, 3]` |
| `Dictionary<K,V>` | `map[K]V` | object | `{"key": "value"}` |

## Versioning Strategy

### Message Type Versioning
- Include version in type name: `CreateOrderCommandV2`
- Maintain backward compatibility
- Support multiple versions simultaneously

### Field Evolution
1. **Adding fields**: Optional with defaults
2. **Removing fields**: Deprecate first
3. **Renaming fields**: Add new, deprecate old
4. **Changing types**: New message version

## Security Considerations

### Sensitive Data
- Never include passwords in messages
- Mask sensitive fields in logs
- Use headers for auth tokens
- Encrypt payloads if needed

### Message Signing
Optional HMAC signature in headers:
- Header: `x-signature`
- Algorithm: HMAC-SHA256
- Key: Shared secret

## Performance Guidelines

### Message Size
- Keep under 64KB for optimal performance
- Use references for large data
- Compress if over 1MB

### Batching
- Group related messages
- Use same correlation ID
- Limit batch size to 100 messages

## Compatibility Testing

### Test Cases
1. Round-trip serialization
2. Cross-language message exchange
3. Version compatibility
4. Error handling
5. Special characters and encoding

### Validation Tools
- JSON Schema validation
- Contract testing
- Integration test suites
- Message replay tools

## Migration Guide

### From Custom Format
1. Map existing fields to envelope
2. Update serialization code
3. Test with both implementations
4. Deploy incrementally

### Between Versions
1. Deploy consumers first
2. Update producers
3. Monitor for errors
4. Remove old version support

## Examples

### Full Message Flow

1. **Client sends command**
```json
{
  "id": "100e8400-e29b-41d4-a716-446655440005",
  "type": "ProcessPaymentCommand",
  "timestamp": "2024-01-15T10:35:00Z",
  "correlationId": "order-789",
  "replyTo": "rpl.100e8400-e29b-41d4-a716-446655440005",
  "headers": {
    "userId": "user-123",
    "priority": "high"
  },
  "payload": {
    "orderId": "ORDER-789",
    "amount": 99.99,
    "currency": "USD",
    "paymentMethod": "credit-card"
  }
}
```

2. **Service publishes event**
```json
{
  "id": "200e8400-e29b-41d4-a716-446655440006",
  "type": "PaymentProcessedEvent",
  "timestamp": "2024-01-15T10:35:05Z",
  "correlationId": "order-789",
  "headers": {
    "aggregateId": "PAY-456",
    "causationId": "100e8400-e29b-41d4-a716-446655440005"
  },
  "payload": {
    "paymentId": "PAY-456",
    "orderId": "ORDER-789",
    "amount": 99.99,
    "status": "Approved",
    "processedAt": "2024-01-15T10:35:05Z"
  }
}
```

3. **Service sends reply**
```json
{
  "id": "300e8400-e29b-41d4-a716-446655440007",
  "type": "PaymentReply",
  "timestamp": "2024-01-15T10:35:05Z",
  "correlationId": "order-789",
  "headers": {
    "requestId": "100e8400-e29b-41d4-a716-446655440005"
  },
  "payload": {
    "success": true,
    "paymentId": "PAY-456",
    "status": "Approved",
    "authorizationCode": "AUTH-123"
  }
}
```

## Reference Implementation

Both .NET and Go implementations provide:
- Envelope serialization/deserialization
- AMQP property mapping
- Queue naming helpers
- Message validation
- Contract testing utilities

See platform-specific documentation for API details.