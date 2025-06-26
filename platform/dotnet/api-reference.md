# .NET API Reference

This document provides a complete API reference for the Mmate .NET framework.

## Table of Contents

- [Contracts](#contracts)
- [Messaging](#messaging)
- [StageFlow](#stageflow)
- [SyncAsyncBridge](#syncasyncbridge)
- [Interceptors](#interceptors)
- [Monitoring](#monitoring)
- [Health](#health)
- [Dependency Injection](#dependency-injection)

## Contracts

### Core Interfaces

#### ICommand
```csharp
public interface ICommand : IMessage
{
    // Marker interface for command messages
}
```

#### IEvent
```csharp
public interface IEvent : IMessage
{
    // Marker interface for event messages
}
```

#### IQuery<TResponse>
```csharp
public interface IQuery<TResponse> : IMessage where TResponse : IResponse
{
    // Marker interface for query messages with response type
}
```

#### IResponse
```csharp
public interface IResponse : IMessage
{
    // Marker interface for response messages
}
```

### Base Classes

#### BaseCommand
```csharp
public abstract class BaseCommand : ICommand
{
    public string MessageId { get; set; } = Guid.NewGuid().ToString();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string CorrelationId { get; set; }
}
```

#### BaseEvent
```csharp
public abstract class BaseEvent : IEvent
{
    public string EventId { get; set; } = Guid.NewGuid().ToString();
    public DateTime OccurredAt { get; set; } = DateTime.UtcNow;
    public string CorrelationId { get; set; }
}
```

#### BaseQuery<TResponse>
```csharp
public abstract class BaseQuery<TResponse> : IQuery<TResponse> where TResponse : IResponse
{
    public string QueryId { get; set; } = Guid.NewGuid().ToString();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}
```

#### BaseResponse
```csharp
public abstract class BaseResponse : IResponse
{
    public bool Success { get; set; }
    public string ErrorMessage { get; set; }
    public string CorrelationId { get; set; }
}
```

## Messaging

### Core Interfaces

#### IMessagePublisher
```csharp
public interface IMessagePublisher
{
    Task PublishAsync<T>(T message, CancellationToken cancellationToken = default) where T : class;
    Task PublishAsync(string routingKey, object message, CancellationToken cancellationToken = default);
}
```

#### IMessageDispatcher
```csharp
public interface IMessageDispatcher : IMessagePublisher
{
    Task<TResponse> RequestAsync<TRequest, TResponse>(
        TRequest request, 
        TimeSpan? timeout = null, 
        CancellationToken cancellationToken = default) 
        where TRequest : class 
        where TResponse : class;
}
```

#### IMessageHandler<T>
```csharp
public interface IMessageHandler<T> where T : class
{
    Task HandleAsync(T message, CancellationToken cancellationToken);
}
```

### Configuration

#### MessagingOptions
```csharp
public class MessagingOptions
{
    public string ConnectionString { get; set; }
    public string ExchangeName { get; set; } = "mmate.exchange";
    public string QueuePrefix { get; set; } = "mmate.";
    public bool EnableAutoAcknowledgment { get; set; } = true;
    public int MaxConcurrentMessages { get; set; } = 10;
}
```

#### RabbitMqConnectionOptions
```csharp
public class RabbitMqConnectionOptions
{
    public string ConnectionString { get; set; }
    public string ClientName { get; set; } = "Mmate.Messaging";
    public bool EnableAutomaticRecovery { get; set; } = true;
    public int NetworkRecoveryIntervalMs { get; set; } = 5000;
    public ushort HeartbeatIntervalSeconds { get; set; } = 60;
    public int ConnectionTimeoutMs { get; set; } = 30000;
    public int MaxRetryAttempts { get; set; } = 5;
    public int InitialRetryDelayMs { get; set; } = 1000;
    public int MaxRetryDelayMs { get; set; } = 30000;
    public double RetryMultiplier { get; set; } = 2.0;
    
    // Circuit breaker settings
    public bool EnableCircuitBreaker { get; set; } = true;
    public int CircuitBreakerFailureThreshold { get; set; } = 5;
    public int CircuitBreakerOpenTimeoutSeconds { get; set; } = 30;
}
```

### Extension Methods

```csharp
public static class ServiceCollectionExtensions
{
    public static IMmateMessagingBuilder AddMmateMessaging(
        this IServiceCollection services,
        Action<MessagingOptions> configure = null);
        
    public static IMmateMessagingBuilder WithRabbitMqTransport(
        this IMmateMessagingBuilder builder,
        Action<RabbitMqConnectionOptions> configure);
        
    public static IMmateMessagingBuilder AddMessageHandler<THandler, TMessage>(
        this IMmateMessagingBuilder builder)
        where THandler : class, IMessageHandler<TMessage>
        where TMessage : class;
}
```

## StageFlow

### Core Interfaces

#### IPipeline<TContext>
```csharp
public interface IPipeline<TContext>
{
    string Name { get; }
    Task ProcessAsync(TContext context, CancellationToken cancellationToken = default);
}
```

#### IPipelineBuilder<TContext>
```csharp
public interface IPipelineBuilder<TContext>
{
    IPipelineBuilder<TContext> AddStage(IStage<TContext> stage);
    IPipelineBuilder<TContext> AddStage<TStage>() where TStage : IStage<TContext>;
    IPipelineBuilder<TContext> AddStage(Func<IServiceProvider, IStage<TContext>> stageFactory);
    IPipelineBuilder<TContext> AddStage(Func<TContext, CancellationToken, Task> stageAction);
    IPipelineBuilder<TContext> AddStageIf(bool condition, IStage<TContext> stage);
    IPipelineBuilder<TContext> WithName(string name);
    IPipeline<TContext> Build();
}
```

#### IStage<TContext>
```csharp
public interface IStage<TContext>
{
    Task ExecuteAsync(TContext context, CancellationToken cancellationToken = default);
}
```

#### IFlowEndpoint<TRequest, TState>
```csharp
public interface IFlowEndpoint<TRequest, TState> where TRequest : class where TState : class
{
    string EndpointId { get; }
    void Stage<TMessage>(Func<IStageContext, TState, TMessage, Task> handler) where TMessage : class;
    void LastStage<TMessage, TResponse>(Func<IStageContext, TState, TMessage, Task<TResponse>> handler) 
        where TMessage : class 
        where TResponse : class;
}
```

### Configuration

#### StageFlowOptions
```csharp
public class StageFlowOptions
{
    public string StageQueuePrefix { get; set; } = "mmate.stageflow.";
    public int MaxStageConcurrency { get; set; } = 5;
    public TimeSpan DefaultStageTimeout { get; set; } = TimeSpan.FromMinutes(5);
}
```

### Extension Methods

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddStageFlow(
        this IServiceCollection services,
        Action<StageFlowOptions> configure = null);
        
    public static IServiceCollection AddFlowEndpointFactory(
        this IServiceCollection services);
}
```

## Interceptors

### Core Interfaces

#### IMessageInterceptor
```csharp
public interface IMessageInterceptor
{
    Task OnBeforePublishAsync(MessageContext context, CancellationToken cancellationToken = default);
    Task OnAfterPublishAsync(MessageContext context, CancellationToken cancellationToken = default);
    Task OnBeforeConsumeAsync(MessageContext context, CancellationToken cancellationToken = default);
    Task OnAfterConsumeAsync(MessageContext context, CancellationToken cancellationToken = default);
}
```

#### IRetryScheduler
```csharp
public interface IRetryScheduler
{
    Task ScheduleRetryAsync(MessageContext context, TimeSpan delay, CancellationToken cancellationToken = default);
}
```

### MessageContext
```csharp
public class MessageContext
{
    public object Message { get; set; }
    public Dictionary<string, object> Headers { get; set; }
    public Dictionary<string, object> Properties { get; set; }
    public string RoutingKey { get; set; }
    public string Exchange { get; set; }
    
    public void AddHeader(string key, object value);
    public T GetHeader<T>(string key, T defaultValue = default);
    public Task RetryAsync(TimeSpan delay, CancellationToken cancellationToken = default);
}
```

### Base Classes

#### MessageInterceptorBase
```csharp
public abstract class MessageInterceptorBase : IMessageInterceptor
{
    public virtual Task OnBeforePublishAsync(MessageContext context, CancellationToken cancellationToken = default) 
        => Task.CompletedTask;
        
    public virtual Task OnAfterPublishAsync(MessageContext context, CancellationToken cancellationToken = default) 
        => Task.CompletedTask;
        
    public virtual Task OnBeforeConsumeAsync(MessageContext context, CancellationToken cancellationToken = default) 
        => Task.CompletedTask;
        
    public virtual Task OnAfterConsumeAsync(MessageContext context, CancellationToken cancellationToken = default) 
        => Task.CompletedTask;
}
```

### Extension Methods

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMmateInterceptors(
        this IServiceCollection services,
        Action<InterceptorRegistry> configure = null);
        
    public static IServiceCollection AddInterceptor<TInterceptor>(
        this IServiceCollection services)
        where TInterceptor : class, IMessageInterceptor;
}
```

## Schema

### Core Interfaces

#### ISchemaRegistry
```csharp
public interface ISchemaRegistry
{
    void RegisterSchema(Type messageType, JsonSchema schema);
    JsonSchema GetSchema(Type messageType);
    bool TryGetSchema(Type messageType, out JsonSchema schema);
}
```

#### ISchemaValidator
```csharp
public interface ISchemaValidator
{
    ValidationResult Validate(object message);
    ValidationResult Validate(object message, JsonSchema schema);
}
```

### Attributes

#### MessageSchemaAttribute
```csharp
[AttributeUsage(AttributeTargets.Class)]
public class MessageSchemaAttribute : Attribute
{
    public string SchemaVersion { get; set; }
    public string Namespace { get; set; }
    public bool StrictValidation { get; set; }
}
```

#### MessagePropertyAttribute
```csharp
[AttributeUsage(AttributeTargets.Property)]
public class MessagePropertyAttribute : Attribute
{
    public bool Required { get; set; }
    public string Description { get; set; }
    public object DefaultValue { get; set; }
}
```

## SyncAsyncBridge

### Core Interfaces

#### IMessagingBridge
```csharp
public interface IMessagingBridge : IDisposable
{
    Task InitializeAsync(CancellationToken cancellationToken = default);
    Task SendAsync(string destination, ReadOnlyMemory<byte> message, CancellationToken cancellationToken = default);
    Task<ReadOnlyMemory<byte>> SendAndReceiveAsync(
        string destination, 
        ReadOnlyMemory<byte> message, 
        int timeoutSeconds = 30, 
        CancellationToken cancellationToken = default);
}
```

### Configuration

#### MessagingBridgeOptions
```csharp
public class MessagingBridgeOptions
{
    public int MaxConcurrentRequests { get; set; } = 1000;
    public string ReplyQueuePrefix { get; set; } = "bridge.reply.";
    public TimeSpan DefaultTimeout { get; set; } = TimeSpan.FromSeconds(30);
    public TimeSpan CleanupInterval { get; set; } = TimeSpan.FromMinutes(5);
    public bool EnableAutoCleanup { get; set; } = true;
    public bool ThrowOnTimeout { get; set; } = true;
}
```

#### RabbitMqMessagingBridgeOptions
```csharp
public class RabbitMqMessagingBridgeOptions : MessagingBridgeOptions
{
    public string ConnectionString { get; set; } = "amqp://guest:guest@localhost:5672/";
    public string ExchangeName { get; set; } = "mmate.bridge";
    public string ExchangeType { get; set; } = "topic";
    public int DefaultTimeoutSeconds { get; set; } = 30;
    public bool DurableExchange { get; set; } = true;
    public bool AutoDeleteQueues { get; set; } = false;
    public int MaxRetryAttempts { get; set; } = 3;
}
```

### Extension Methods

```csharp
public static class MessagingBridgeExtensions
{
    public static Task SendStringAsync(
        this IMessagingBridge bridge,
        string destination,
        string message,
        Encoding encoding = null,
        CancellationToken cancellationToken = default);
        
    public static Task<string> SendAndReceiveStringAsync(
        this IMessagingBridge bridge,
        string destination,
        string message,
        int timeoutSeconds = 30,
        Encoding encoding = null,
        CancellationToken cancellationToken = default);
        
    public static Task SendJsonAsync<T>(
        this IMessagingBridge bridge,
        string destination,
        T message,
        JsonSerializerOptions options = null,
        CancellationToken cancellationToken = default);
        
    public static Task<TResponse> SendAndReceiveJsonAsync<TRequest, TResponse>(
        this IMessagingBridge bridge,
        string destination,
        TRequest message,
        int timeoutSeconds = 30,
        JsonSerializerOptions options = null,
        CancellationToken cancellationToken = default);
}
```

## Monitoring

### Command Line Interface

```bash
# List queues
mmate-monitor queue list [options]

# Show queue details
mmate-monitor queue show <queue-name> [options]

# List messages in queue
mmate-monitor message list <queue-name> [options]

# Show message details
mmate-monitor message show <queue-name> <message-id> [options]

# List dead letter queues
mmate-monitor dlq list [options]

# Requeue messages from DLQ
mmate-monitor dlq requeue <dlq-name> [options]

# Watch queue activity
mmate-monitor watch <queue-name> [options]
```

### Common Options

```
--connection-string, -c    RabbitMQ connection string
--output, -o              Output format (table, json, csv)
--filter, -f              Filter pattern for queue names
--sort, -s                Sort by (name, messages, consumers)
--limit, -l               Limit number of results
--include-empty           Include empty queues
--include-system          Include system queues
```

### Programmatic API

#### IBrokerMonitor
```csharp
public interface IBrokerMonitor : IDisposable, IAsyncDisposable
{
    Task ConnectAsync(string connectionString, CancellationToken cancellationToken = default);
    Task<IEnumerable<QueueInfo>> GetQueuesAsync(
        string filter = null, 
        bool includeEmpty = true, 
        bool includeSystem = false, 
        CancellationToken cancellationToken = default);
    Task<QueueDetailedInfo> GetQueueInfoAsync(string queueName, CancellationToken cancellationToken = default);
    Task<IEnumerable<MessageInfo>> GetMessagesAsync(
        string queueName, 
        int limit = 10, 
        CancellationToken cancellationToken = default);
    Task<MessageDetailedInfo> GetMessageInfoAsync(
        string queueName, 
        string messageId, 
        string correlationId = null, 
        CancellationToken cancellationToken = default);
    Task<int> PurgeQueueAsync(string queueName, CancellationToken cancellationToken = default);
    Task RequeueMessageAsync(
        string sourceQueue, 
        string targetQueue, 
        string messageId, 
        CancellationToken cancellationToken = default);
    Task<int> RequeueAllMessagesAsync(
        string sourceQueue, 
        string targetQueue, 
        CancellationToken cancellationToken = default);
    Task<IEnumerable<QueueInfo>> GetDeadLetterQueuesAsync(CancellationToken cancellationToken = default);
    Task CloseAsync();
}
```

## Health

### Health Checks

```csharp
public class MmateHealthCheck : IHealthCheck
{
    private readonly IConnectionManager _connectionManager;
    
    public MmateHealthCheck(IConnectionManager connectionManager)
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context,
        CancellationToken cancellationToken = default)
}

// Registration
services.AddHealthChecks()
    .AddRabbitMQ(rabbitConnectionString: "amqp://localhost", name: "rabbitmq")
    .AddCheck<MmateHealthCheck>("mmate", tags: new[] { "messaging" });
```

### Health Check Results

```csharp
public class HealthReport
{
    public HealthStatus Status { get; set; }
    public TimeSpan TotalDuration { get; set; }
    public Dictionary<string, HealthReportEntry> Entries { get; set; }
}

public class HealthReportEntry
{
    public HealthStatus Status { get; set; }
    public string Description { get; set; }
    public TimeSpan Duration { get; set; }
    public Exception Exception { get; set; }
    public IReadOnlyDictionary<string, object> Data { get; set; }
}
```

## Dependency Injection

### Service Registration

```csharp
public static class ServiceCollectionExtensions
{
    // Add Mmate with configuration
    public static IServiceCollection AddMmate(this IServiceCollection services,
        Action<MmateOptions> configure)
    
    // Add Mmate with IConfiguration
    public static IServiceCollection AddMmate(this IServiceCollection services,
        IConfiguration configuration)
    
    // Add message handlers from assembly
    public static IServiceCollection AddMmateHandlers(this IServiceCollection services,
        params Assembly[] assemblies)
    
    // Add interceptors
    public static IServiceCollection AddMmateInterceptor<TInterceptor>(
        this IServiceCollection services) where TInterceptor : class, IMessageInterceptor
}

// Configuration
public class MmateOptions
{
    public string ConnectionString { get; set; }
    public int PrefetchCount { get; set; } = 10;
    public RetryPolicy RetryPolicy { get; set; }
    public List<Type> Interceptors { get; set; } = new();
    public Dictionary<string, ConsumerGroupConfiguration> ConsumerGroups { get; set; } = new();
}
```

### Application Builder Extensions

```csharp
public static class ApplicationBuilderExtensions
{
    // Use Mmate middleware
    public static IApplicationBuilder UseMmate(this IApplicationBuilder app)
    
    // Map health checks
    public static IEndpointRouteBuilder MapMmateHealthChecks(
        this IEndpointRouteBuilder endpoints, string pattern = "/health")
}
```

### Hosted Service

```csharp
public class MmateHostedService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly MmateOptions _options;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
}
```

## Attributes

### Message Handler Attributes

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class MessageHandlerAttribute : Attribute
{
    public string Queue { get; set; }
    public int PrefetchCount { get; set; }
    public bool AutoAck { get; set; }
}

[AttributeUsage(AttributeTargets.Class)]
public class ConsumerGroupAttribute : Attribute
{
    public string Name { get; set; }
    public int Size { get; set; }
    
    public ConsumerGroupAttribute(string name, int size = 5)
}

[AttributeUsage(AttributeTargets.Class)]
public class RetryPolicyAttribute : Attribute
{
    public int MaxAttempts { get; set; }
    public int InitialDelayMs { get; set; }
    public BackoffStrategy Strategy { get; set; }
}
```

## Common Types

### Retry Policy

```csharp
public class RetryPolicy
{
    public int MaxAttempts { get; set; } = 3;
    public TimeSpan InitialDelay { get; set; } = TimeSpan.FromSeconds(1);
    public TimeSpan MaxDelay { get; set; } = TimeSpan.FromMinutes(1);
    public BackoffStrategy Strategy { get; set; } = BackoffStrategy.Exponential;
}

public enum BackoffStrategy
{
    Linear,
    Exponential,
    Constant
}

public class RetryPolicyBuilder
{
    public RetryPolicyBuilder MaxAttempts(int attempts)
    public RetryPolicyBuilder WithLinearBackoff(TimeSpan initialDelay)
    public RetryPolicyBuilder WithExponentialBackoff(TimeSpan initialDelay = default)
    public RetryPolicyBuilder WithConstantDelay(TimeSpan delay)
}
```

### Message Extensions

```csharp
public static class MessageExtensions
{
    public static string GetRoutingKey(this IMessage message)
    public static void AddHeader(this IMessage message, string key, string value)
    public static string GetHeader(this IMessage message, string key)
    public static bool TryGetHeader(this IMessage message, string key, out string value)
}
```

### Context Extensions

```csharp
public static class MessageContextExtensions
{
    public static void SetRetryCount(this MessageContext context, int count)
    public static int GetRetryCount(this MessageContext context)
    public static void SetDeadLetterReason(this MessageContext context, string reason)
    public static string GetDeadLetterReason(this MessageContext context)
}
```

## Error Handling

### Common Exceptions

```csharp
// Circuit breaker opened
public class CircuitBreakerOpenException : Exception
{
    public CircuitBreakerOpenException(string message) : base(message) { }
}

// General messaging exception
public class MessagingException : Exception
{
    public string QueueName { get; set; }
    public string MessageId { get; set; }
    public MessagingException(string message, Exception innerException = null) 
        : base(message, innerException) { }
}

// Workflow execution exception
public class WorkflowException : Exception
{
    public string WorkflowId { get; set; }
    public string StageName { get; set; }
    public WorkflowException(string message, Exception innerException = null) 
        : base(message, innerException) { }
}

// Message timeout exception
public class MessageTimeoutException : Exception
{
    public string CorrelationId { get; set; }
    public TimeSpan Timeout { get; set; }
    public MessageTimeoutException(string message) : base(message) { }
}

// Schema validation exception
public class SchemaValidationException : Exception
{
    public List<string> ValidationErrors { get; set; }
    public SchemaValidationException(string message, List<string> errors) 
        : base(message) 
    {
        ValidationErrors = errors;
    }
}
```

## Best Practices

### Message Design

1. **Keep messages small and focused**
   ```csharp
   // Good
   public class UpdateOrderStatusCommand : Command
   {
       public string OrderId { get; set; }
       public OrderStatus NewStatus { get; set; }
   }
   
   // Avoid
   public class UpdateOrderCommand : Command
   {
       public Order EntireOrder { get; set; }
   }
   ```

2. **Use value objects for complex data**
   ```csharp
   public class Address
   {
       public string Street { get; set; }
       public string City { get; set; }
       public string PostalCode { get; set; }
       public string Country { get; set; }
   }
   ```

3. **Include correlation IDs for tracing**
   ```csharp
   public class OrderEvent : Event
   {
       public OrderEvent(string correlationId) : base("OrderEvent")
       {
           CorrelationId = correlationId;
       }
   }
   ```

4. **Version messages for backward compatibility**
   ```csharp
   [MessageSchema(SchemaVersion = "1.0")]
   public class CreateOrderCommandV1 : Command
   {
       // Version 1 properties
   }
   
   [MessageSchema(SchemaVersion = "2.0")]
   public class CreateOrderCommandV2 : Command
   {
       // Version 2 properties with backward compatibility
   }
   ```

### Error Handling

1. **Use retry policies for transient failures**
   ```csharp
   services.AddMmate(options =>
   {
       options.RetryPolicy = new RetryPolicy
       {
           MaxAttempts = 3,
           Strategy = BackoffStrategy.Exponential,
           InitialDelay = TimeSpan.FromSeconds(1)
       };
   });
   ```

2. **Implement dead letter queues for permanent failures**
   ```csharp
   [MessageHandler(Queue = "orders", DeadLetterQueue = "orders.dlq")]
   public class OrderHandler : IMessageHandler<CreateOrderCommand>
   {
       public async Task HandleAsync(CreateOrderCommand command, MessageContext context)
       {
           if (!IsValid(command))
           {
               throw new PermanentException("Invalid order data");
           }
       }
   }
   ```

3. **Log errors with context for debugging**
   ```csharp
   _logger.LogError(ex, "Failed to process order {OrderId} for customer {CustomerId}",
       command.OrderId, command.CustomerId);
   ```

4. **Monitor circuit breaker states**
   ```csharp
   services.AddHealthChecks()
       .AddCheck("circuit-breaker", () =>
       {
           var state = _circuitBreaker.State;
           return state == CircuitState.Open
               ? HealthCheckResult.Unhealthy("Circuit breaker is open")
               : HealthCheckResult.Healthy();
       });
   ```

### Performance

1. **Use batching for high-volume scenarios**
   ```csharp
   public class BatchOrderProcessor : IMessageHandler<List<CreateOrderCommand>>
   {
       public async Task HandleAsync(List<CreateOrderCommand> commands, MessageContext context)
       {
           await _orderService.CreateOrdersBatchAsync(commands);
       }
   }
   ```

2. **Configure appropriate concurrency limits**
   ```csharp
   services.AddMmate(options =>
   {
       options.MaxConcurrentMessages = Environment.ProcessorCount * 2;
       options.PrefetchCount = 50;
   });
   ```

3. **Monitor queue depths and processing times**
   ```csharp
   public class MetricsInterceptor : IMessageInterceptor
   {
       public async Task OnAfterConsumeAsync(MessageContext context, CancellationToken cancellationToken)
       {
           _metrics.RecordProcessingTime(context.MessageType, context.ProcessingDuration);
           _metrics.RecordQueueDepth(context.QueueName, context.QueueDepth);
       }
   }
   ```

4. **Use connection pooling**
   ```csharp
   services.AddMmate(options =>
   {
       options.ConnectionPoolSize = 10;
       options.ChannelPoolSize = 20;
   });
   ```

### Testing

1. **Use in-memory transport for unit tests**
   ```csharp
   services.AddMmate(options =>
   {
       options.UseInMemoryTransport();
   });
   ```

2. **Mock external dependencies**
   ```csharp
   var mockPublisher = new Mock<IMessagePublisher>();
   mockPublisher
       .Setup(x => x.PublishAsync(It.IsAny<OrderCreatedEvent>(), It.IsAny<CancellationToken>()))
       .Returns(Task.CompletedTask);
   ```

3. **Test error scenarios and retries**
   ```csharp
   [Fact]
   public async Task Should_Retry_On_Transient_Error()
   {
       // Arrange
       var handler = new Mock<IMessageHandler<TestCommand>>();
       handler.SetupSequence(x => x.HandleAsync(It.IsAny<TestCommand>(), It.IsAny<MessageContext>()))
           .ThrowsAsync(new TransientException("Network error"))
           .ThrowsAsync(new TransientException("Network error"))
           .Returns(Task.CompletedTask);
       
       // Act & Assert
       await _subscriber.ProcessMessageAsync(new TestCommand());
       handler.Verify(x => x.HandleAsync(It.IsAny<TestCommand>(), It.IsAny<MessageContext>()), Times.Exactly(3));
   }
   ```

4. **Verify message schema compliance**
   ```csharp
   [Fact]
   public void Should_Validate_Message_Schema()
   {
       var message = new CreateOrderCommand
       {
           // Missing required fields
       };
       
       var result = _schemaValidator.Validate(message);
       
       Assert.False(result.IsValid);
       Assert.Contains("CustomerId is required", result.Errors);
   }
   ```