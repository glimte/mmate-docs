# Migration Guides

These guides help you migrate between different Mmate implementations.

## Available Guides

### [.NET to Go Migration](dotnet-to-go.md)
For teams moving from the .NET implementation to Go:
- API mapping
- Pattern translations
- Common pitfalls
- Step-by-step migration

### [Go to .NET Migration](go-to-dotnet.md)
For teams moving from Go to the .NET implementation:
- Framework differences
- Dependency injection setup
- Async pattern conversion
- Tool equivalents

## General Migration Strategy

### 1. Assessment Phase
- Inventory current usage
- Identify dependencies
- Map message types
- Document custom patterns

### 2. Planning Phase
- Choose migration approach (big bang vs gradual)
- Set up parallel environments
- Plan rollback strategy
- Schedule migration windows

### 3. Implementation Phase
- Set up new environment
- Migrate message contracts
- Port handlers and logic
- Update configurations

### 4. Testing Phase
- Unit test migrations
- Integration testing
- Performance comparison
- Load testing

### 5. Cutover Phase
- Deploy consumers first
- Switch producers
- Monitor closely
- Keep rollback ready

## Cross-Language Compatibility

Both implementations use the same:
- Wire format (JSON)
- Queue naming conventions
- Message routing patterns
- AMQP properties

This ensures messages published by one implementation can be consumed by the other.

## Tools and Utilities

### Message Replay Tool
Replay messages from one system to another:
```bash
mmate replay --source=amqp://old --dest=amqp://new --queue=orders
```

### Contract Validator
Validate message compatibility:
```bash
mmate validate --dotnet=contracts.dll --go=contracts.go
```

### Performance Comparison
Compare throughput between implementations:
```bash
mmate benchmark --impl=dotnet --impl=go --duration=5m
```

## Common Challenges

1. **Language Idioms**
   - Async/await vs goroutines
   - Exception handling vs error returns
   - Dependency injection differences

2. **Framework Features**
   - Attribute-based routing (.NET)
   - Explicit registration (Go)

3. **Configuration**
   - appsettings.json vs environment variables
   - Configuration providers vs structs

4. **Testing**
   - xUnit/NUnit vs Go testing
   - Mocking approaches
   - Integration test setup

## Support

- Check platform-specific documentation
- Review examples in both languages
- Test thoroughly before cutover
- Monitor after migration