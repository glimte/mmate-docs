# Interceptors - Cross-Cutting Concerns

Interceptors provide a powerful mechanism for implementing cross-cutting concerns in your messaging pipeline, such as logging, metrics, tracing, validation, and security.

## Overview

Interceptors allow you to:
- Add behavior before and after message processing
- Modify messages in flight
- Short-circuit processing when needed
- Chain multiple interceptors together
- Apply concerns uniformly across all messages

## Core Concepts

### Interceptor Pipeline
```
Message → Interceptor 1 → Interceptor 2 → ... → Handler
        ←              ←               ← ... ←
```

### Execution Order
- **Pre-processing**: Interceptors execute in the order they were added
- **Post-processing**: Interceptors execute in reverse order
- **Short-circuit**: Any interceptor can stop the pipeline

## Basic Usage

### Creating Interceptors

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Simple interceptor
public class LoggingInterceptor : IMessageInterceptor
{
    private readonly ILogger<LoggingInterceptor> _logger;
    
    public LoggingInterceptor(
        ILogger<LoggingInterceptor> logger)
    {
        _logger = logger;
    }
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        // Pre-processing
        _logger.LogInformation(
            "Processing {MessageType} with ID {MessageId}",
            context.Message.GetType().Name,
            context.MessageId);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            // Call next interceptor/handler
            context = await next(context);
            
            // Post-processing
            _logger.LogInformation(
                "Processed {MessageType} in {Duration}ms",
                context.Message.GetType().Name,
                stopwatch.ElapsedMilliseconds);
            
            return context;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Error processing {MessageType}",
                context.Message.GetType().Name);
            throw;
        }
    }
}
```

</td>
<td>

```go
// Simple interceptor
type LoggingInterceptor struct {
    logger *slog.Logger
}

func (i *LoggingInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Pre-processing
    i.logger.Info("Processing message",
        "type", msg.GetType(),
        "id", msg.GetID())
    
    start := time.Now()
    
    // Call next interceptor/handler
    err := next(ctx, msg)
    
    // Post-processing
    duration := time.Since(start)
    if err != nil {
        i.logger.Error("Error processing message",
            "type", msg.GetType(),
            "duration", duration,
            "error", err)
    } else {
        i.logger.Info("Processed message",
            "type", msg.GetType(),
            "duration", duration)
    }
    
    return err
}
```

</td>
</tr>
</table>

### Registering Interceptors

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Register globally
services.AddMmate(options =>
{
    options.AddInterceptor<LoggingInterceptor>();
    options.AddInterceptor<MetricsInterceptor>();
    options.AddInterceptor<TracingInterceptor>();
    options.AddInterceptor<ValidationInterceptor>();
});

// Register for specific message types
services.AddMmate(options =>
{
    options.AddInterceptor<AuthorizationInterceptor>(
        filter => filter
            .ForCommands()
            .ForMessageType<CreateOrderCommand>());
    
    options.AddInterceptor<AuditInterceptor>(
        filter => filter
            .ForEvents()
            .WithPredicate(msg => 
                msg.Headers.ContainsKey("audit")));
});

// Order matters!
services.AddMmate(options =>
{
    // Executes first
    options.AddInterceptor<SecurityInterceptor>();
    // Executes second
    options.AddInterceptor<ValidationInterceptor>();
    // Executes third
    options.AddInterceptor<LoggingInterceptor>();
});
```

</td>
<td>

```go
// Create pipeline
pipeline := interceptors.NewPipeline()

// Register globally
pipeline.Use(&LoggingInterceptor{logger: logger})
pipeline.Use(&MetricsInterceptor{metrics: metrics})
pipeline.Use(&TracingInterceptor{tracer: tracer})
pipeline.Use(&ValidationInterceptor{})

// Register with filters
pipeline.Use(&AuthorizationInterceptor{},
    interceptors.ForMessageTypes(
        "CreateOrderCommand",
        "UpdateOrderCommand"))

pipeline.Use(&AuditInterceptor{},
    interceptors.ForEvents(),
    interceptors.WithPredicate(
        func(msg contracts.Message) bool {
            return msg.GetHeader("audit") != ""
        }))

// Order matters!
pipeline = interceptors.NewPipeline(
    &SecurityInterceptor{},    // First
    &ValidationInterceptor{},  // Second  
    &LoggingInterceptor{},    // Third
)

// Apply to subscriber
subscriber := messaging.NewSubscriber(
    transport,
    messaging.WithInterceptors(pipeline))
```

</td>
</tr>
</table>

## Common Interceptors

### Logging Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class StructuredLoggingInterceptor : 
    IMessageInterceptor
{
    private readonly ILogger _logger;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["MessageId"] = context.MessageId,
            ["MessageType"] = context.Message.GetType().Name,
            ["CorrelationId"] = context.CorrelationId,
            ["UserId"] = context.Headers.GetValueOrDefault("userId"),
            ["TenantId"] = context.Headers.GetValueOrDefault("tenantId")
        }))
        {
            _logger.LogInformation("Message received");
            
            try
            {
                context = await next(context);
                _logger.LogInformation("Message processed");
                return context;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Message processing failed");
                throw;
            }
        }
    }
}
```

</td>
<td>

```go
type StructuredLoggingInterceptor struct {
    logger *slog.Logger
}

func (i *StructuredLoggingInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Add structured fields
    logger := i.logger.With(
        "messageId", msg.GetID(),
        "messageType", msg.GetType(),
        "correlationId", msg.GetCorrelationID(),
        "userId", msg.GetHeader("userId"),
        "tenantId", msg.GetHeader("tenantId"))
    
    logger.Info("Message received")
    
    err := next(ctx, msg)
    
    if err != nil {
        logger.Error("Message processing failed",
            "error", err)
    } else {
        logger.Info("Message processed")
    }
    
    return err
}
```

</td>
</tr>
</table>

### Metrics Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class MetricsInterceptor : IMessageInterceptor
{
    private readonly IMetrics _metrics;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        var messageType = context.Message.GetType().Name;
        var timer = _metrics.StartTimer(
            "message.processing.duration",
            new { type = messageType });
        
        _metrics.Increment("message.received",
            new { type = messageType });
        
        try
        {
            context = await next(context);
            
            _metrics.Increment("message.processed",
                new { type = messageType, status = "success" });
            
            return context;
        }
        catch (Exception ex)
        {
            _metrics.Increment("message.processed",
                new { type = messageType, status = "error" });
            
            _metrics.Increment("message.errors",
                new { 
                    type = messageType, 
                    error = ex.GetType().Name 
                });
            
            throw;
        }
        finally
        {
            timer.Stop();
        }
    }
}
```

</td>
<td>

```go
type MetricsInterceptor struct {
    metrics Metrics
}

func (i *MetricsInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    messageType := msg.GetType()
    
    // Start timer
    start := time.Now()
    defer func() {
        i.metrics.RecordDuration(
            "message.processing.duration",
            time.Since(start),
            "type", messageType)
    }()
    
    // Count received
    i.metrics.Increment("message.received",
        "type", messageType)
    
    // Process
    err := next(ctx, msg)
    
    // Count result
    if err != nil {
        i.metrics.Increment("message.processed",
            "type", messageType,
            "status", "error")
        
        i.metrics.Increment("message.errors",
            "type", messageType,
            "error", fmt.Sprintf("%T", err))
    } else {
        i.metrics.Increment("message.processed",
            "type", messageType,
            "status", "success")
    }
    
    return err
}
```

</td>
</tr>
</table>

### Tracing Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class TracingInterceptor : IMessageInterceptor
{
    private readonly ITracer _tracer;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        // Extract parent context from headers
        var parentContext = _tracer.Extract(
            BuiltinFormats.TextMap,
            new TextMapExtractAdapter(context.Headers));
        
        // Start new span
        using var scope = _tracer.BuildSpan(
            $"process.{context.Message.GetType().Name}")
            .AsChildOf(parentContext)
            .WithTag("message.id", context.MessageId)
            .WithTag("message.type", context.Message.GetType().Name)
            .StartActive(finishSpanOnDispose: true);
        
        var span = scope.Span;
        
        try
        {
            // Inject span context for downstream
            var headers = new Dictionary<string, string>();
            _tracer.Inject(
                span.Context,
                BuiltinFormats.TextMap,
                new TextMapInjectAdapter(headers));
            
            foreach (var header in headers)
            {
                context.Headers[header.Key] = header.Value;
            }
            
            return await next(context);
        }
        catch (Exception ex)
        {
            span.SetTag("error", true);
            span.Log(new Dictionary<string, object>
            {
                ["event"] = "error",
                ["error.kind"] = ex.GetType().Name,
                ["message"] = ex.Message,
                ["stack"] = ex.StackTrace
            });
            throw;
        }
    }
}
```

</td>
<td>

```go
type TracingInterceptor struct {
    tracer trace.Tracer
}

func (i *TracingInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Extract parent context from headers
    carrier := &HeaderCarrier{msg: msg}
    ctx = otel.GetTextMapPropagator().Extract(
        ctx, carrier)
    
    // Start new span
    ctx, span := i.tracer.Start(ctx,
        fmt.Sprintf("process.%s", msg.GetType()),
        trace.WithAttributes(
            attribute.String("message.id", msg.GetID()),
            attribute.String("message.type", msg.GetType()),
        ))
    defer span.End()
    
    // Inject span context for downstream
    otel.GetTextMapPropagator().Inject(ctx, carrier)
    
    // Process
    err := next(ctx, msg)
    
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
    } else {
        span.SetStatus(codes.Ok, "")
    }
    
    return err
}

// HeaderCarrier implements TextMapCarrier
type HeaderCarrier struct {
    msg contracts.Message
}

func (c *HeaderCarrier) Get(key string) string {
    return c.msg.GetHeader(key)
}

func (c *HeaderCarrier) Set(key, value string) {
    c.msg.SetHeader(key, value)
}

func (c *HeaderCarrier) Keys() []string {
    return c.msg.GetHeaderKeys()
}
```

</td>
</tr>
</table>

### Validation Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ValidationInterceptor : IMessageInterceptor
{
    private readonly IValidator _validator;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        // Skip validation for replies
        if (context.Message is IReply)
        {
            return await next(context);
        }
        
        // Validate message
        var validationResult = await _validator
            .ValidateAsync(context.Message);
        
        if (!validationResult.IsValid)
        {
            throw new ValidationException(
                "Message validation failed",
                validationResult.Errors);
        }
        
        // Validate based on type
        switch (context.Message)
        {
            case ICommand command:
                ValidateCommand(command);
                break;
            case IEvent @event:
                ValidateEvent(@event);
                break;
            case IQuery query:
                ValidateQuery(query);
                break;
        }
        
        return await next(context);
    }
    
    private void ValidateCommand(ICommand command)
    {
        if (string.IsNullOrEmpty(command.CommandId))
            throw new ValidationException("CommandId required");
        
        if (command.Timestamp > DateTime.UtcNow.AddMinutes(5))
            throw new ValidationException("Command from future");
    }
}
```

</td>
<td>

```go
type ValidationInterceptor struct {
    validator Validator
}

func (i *ValidationInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Skip validation for replies
    if _, isReply := msg.(contracts.Reply); isReply {
        return next(ctx, msg)
    }
    
    // Validate message
    if err := i.validator.Validate(msg); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }
    
    // Type-specific validation
    switch m := msg.(type) {
    case contracts.Command:
        if err := i.validateCommand(m); err != nil {
            return err
        }
    case contracts.Event:
        if err := i.validateEvent(m); err != nil {
            return err
        }
    case contracts.Query:
        if err := i.validateQuery(m); err != nil {
            return err
        }
    }
    
    return next(ctx, msg)
}

func (i *ValidationInterceptor) validateCommand(
    cmd contracts.Command) error {
    
    if cmd.GetID() == "" {
        return errors.New("command ID required")
    }
    
    if cmd.GetTimestamp().After(
        time.Now().Add(5 * time.Minute)) {
        return errors.New("command from future")
    }
    
    return nil
}
```

</td>
</tr>
</table>

### Authorization Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class AuthorizationInterceptor : IMessageInterceptor
{
    private readonly IAuthorizationService _authService;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        // Extract user context
        var userId = context.Headers.GetValueOrDefault("userId");
        var tenantId = context.Headers.GetValueOrDefault("tenantId");
        var roles = context.Headers
            .GetValueOrDefault("roles")?.Split(',') ?? Array.Empty<string>();
        
        var userContext = new UserContext(userId, tenantId, roles);
        
        // Check authorization
        var authResult = await _authService.AuthorizeAsync(
            userContext,
            context.Message);
        
        if (!authResult.Succeeded)
        {
            throw new UnauthorizedException(
                $"User {userId} not authorized for {context.Message.GetType().Name}",
                authResult.FailureReasons);
        }
        
        // Add user context for downstream
        context.Items["UserContext"] = userContext;
        
        return await next(context);
    }
}

// Usage in handler
public class OrderHandler : IMessageHandler<CreateOrderCommand>
{
    public async Task HandleAsync(
        CreateOrderCommand command,
        MessageContext context)
    {
        var userContext = context.Items["UserContext"] as UserContext;
        
        // Use user context for business logic
        if (!userContext.HasRole("OrderCreator"))
        {
            throw new BusinessException("Insufficient permissions");
        }
    }
}
```

</td>
<td>

```go
type AuthorizationInterceptor struct {
    authService AuthorizationService
}

func (i *AuthorizationInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Extract user context
    userID := msg.GetHeader("userId")
    tenantID := msg.GetHeader("tenantId")
    rolesStr := msg.GetHeader("roles")
    
    var roles []string
    if rolesStr != "" {
        roles = strings.Split(rolesStr, ",")
    }
    
    userCtx := &UserContext{
        UserID:   userID,
        TenantID: tenantID,
        Roles:    roles,
    }
    
    // Check authorization
    allowed, err := i.authService.Authorize(
        ctx, userCtx, msg)
    if err != nil {
        return fmt.Errorf("authorization check failed: %w", err)
    }
    
    if !allowed {
        return &UnauthorizedError{
            UserID:      userID,
            MessageType: msg.GetType(),
        }
    }
    
    // Add user context for downstream
    ctx = context.WithValue(ctx, userContextKey, userCtx)
    
    return next(ctx, msg)
}

// Usage in handler
func (h *OrderHandler) Handle(
    ctx context.Context,
    msg contracts.Message) error {
    
    userCtx := ctx.Value(userContextKey).(*UserContext)
    
    // Use user context for business logic
    if !userCtx.HasRole("OrderCreator") {
        return errors.New("insufficient permissions")
    }
    
    // Process order...
}
```

</td>
</tr>
</table>

## Advanced Patterns

### Conditional Interceptors

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ConditionalInterceptor : IMessageInterceptor
{
    private readonly IMessageInterceptor _inner;
    private readonly Func<MessageContext, bool> _condition;
    
    public ConditionalInterceptor(
        IMessageInterceptor inner,
        Func<MessageContext, bool> condition)
    {
        _inner = inner;
        _condition = condition;
    }
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        if (_condition(context))
        {
            return await _inner.InterceptAsync(context, next);
        }
        
        return await next(context);
    }
}

// Usage
services.AddMmate(options =>
{
    options.AddInterceptor(
        new ConditionalInterceptor(
            new ExpensiveValidationInterceptor(),
            ctx => ctx.Headers.ContainsKey("validate-deep")));
});
```

</td>
<td>

```go
type ConditionalInterceptor struct {
    inner     interceptors.Interceptor
    condition func(contracts.Message) bool
}

func (i *ConditionalInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    if i.condition(msg) {
        return i.inner.Intercept(ctx, msg, next)
    }
    
    return next(ctx, msg)
}

// Usage
pipeline.Use(&ConditionalInterceptor{
    inner: &ExpensiveValidationInterceptor{},
    condition: func(msg contracts.Message) bool {
        return msg.GetHeader("validate-deep") != ""
    },
})
```

</td>
</tr>
</table>

### Circuit Breaker Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class CircuitBreakerInterceptor : IMessageInterceptor
{
    private readonly ICircuitBreaker _circuitBreaker;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        return await _circuitBreaker.ExecuteAsync(
            async () => await next(context),
            context.Message.GetType().Name);
    }
}

// Configure circuit breaker
services.AddSingleton<ICircuitBreaker>(sp =>
{
    return new CircuitBreaker(options =>
    {
        options.FailureThreshold = 5;
        options.SamplingDuration = TimeSpan.FromMinutes(1);
        options.MinimumThroughput = 10;
        options.DurationOfBreak = TimeSpan.FromSeconds(30);
        
        options.OnOpen = (messageType, duration) =>
        {
            logger.LogWarning(
                "Circuit opened for {MessageType} for {Duration}",
                messageType, duration);
        };
        
        options.OnReset = (messageType) =>
        {
            logger.LogInformation(
                "Circuit reset for {MessageType}",
                messageType);
        };
    });
});
```

</td>
<td>

```go
type CircuitBreakerInterceptor struct {
    breakers map[string]*circuit.Breaker
    mu       sync.RWMutex
    options  CircuitBreakerOptions
}

func (i *CircuitBreakerInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    breaker := i.getBreakerForType(msg.GetType())
    
    return breaker.Execute(func() error {
        return next(ctx, msg)
    })
}

func (i *CircuitBreakerInterceptor) getBreakerForType(
    msgType string) *circuit.Breaker {
    
    i.mu.RLock()
    breaker, exists := i.breakers[msgType]
    i.mu.RUnlock()
    
    if exists {
        return breaker
    }
    
    i.mu.Lock()
    defer i.mu.Unlock()
    
    // Double-check
    if breaker, exists := i.breakers[msgType]; exists {
        return breaker
    }
    
    // Create new breaker
    breaker = circuit.NewBreaker(circuit.Options{
        FailureThreshold: i.options.FailureThreshold,
        SuccessThreshold: i.options.SuccessThreshold,
        Timeout:         i.options.Timeout,
        OnStateChange: func(from, to circuit.State) {
            log.Printf("Circuit breaker for %s: %s -> %s",
                msgType, from, to)
        },
    })
    
    i.breakers[msgType] = breaker
    return breaker
}
```

</td>
</tr>
</table>

### Retry Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class RetryInterceptor : IMessageInterceptor
{
    private readonly IRetryPolicy _retryPolicy;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        return await _retryPolicy.ExecuteAsync(
            async () => await next(context),
            (exception, attempt, delay) =>
            {
                _logger.LogWarning(
                    "Retry attempt {Attempt} after {Delay}ms for {MessageType}",
                    attempt,
                    delay.TotalMilliseconds,
                    context.Message.GetType().Name);
                
                // Add retry metadata
                context.Headers[$"retry-attempt"] = attempt.ToString();
                context.Headers[$"retry-delay"] = delay.TotalMilliseconds.ToString();
            });
    }
}

// Configure retry policy
services.AddSingleton<IRetryPolicy>(sp =>
{
    return new ExponentialBackoffRetry(options =>
    {
        options.MaxAttempts = 3;
        options.InitialDelay = TimeSpan.FromSeconds(1);
        options.MaxDelay = TimeSpan.FromSeconds(30);
        options.BackoffMultiplier = 2;
        
        options.ShouldRetry = (exception) =>
        {
            return exception is TransientException ||
                   exception is TimeoutException ||
                   (exception is HttpRequestException httpEx && 
                    IsRetryableStatusCode(httpEx));
        };
    });
});
```

</td>
<td>

```go
type RetryInterceptor struct {
    policy RetryPolicy
    logger *slog.Logger
}

func (i *RetryInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    var lastErr error
    
    for attempt := 0; attempt <= i.policy.MaxAttempts; attempt++ {
        if attempt > 0 {
            delay := i.policy.CalculateDelay(attempt)
            i.logger.Warn("Retrying message processing",
                "attempt", attempt,
                "delay", delay,
                "messageType", msg.GetType(),
                "error", lastErr)
            
            // Add retry metadata
            msg.SetHeader("retry-attempt", 
                strconv.Itoa(attempt))
            msg.SetHeader("retry-delay",
                strconv.FormatInt(delay.Milliseconds(), 10))
            
            select {
            case <-time.After(delay):
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        
        err := next(ctx, msg)
        if err == nil {
            return nil
        }
        
        if !i.policy.ShouldRetry(err) {
            return err
        }
        
        lastErr = err
    }
    
    return fmt.Errorf("max retries exceeded: %w", lastErr)
}
```

</td>
</tr>
</table>

### Caching Interceptor

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class CachingInterceptor : IMessageInterceptor
{
    private readonly ICache _cache;
    private readonly ICacheKeyGenerator _keyGenerator;
    
    public async Task<MessageContext> InterceptAsync(
        MessageContext context,
        MessageDelegate next)
    {
        // Only cache queries
        if (!(context.Message is IQuery))
        {
            return await next(context);
        }
        
        var cacheKey = _keyGenerator.GenerateKey(context.Message);
        var cached = await _cache.GetAsync<object>(cacheKey);
        
        if (cached != null)
        {
            context.Items["CachedResponse"] = cached;
            context.Items["CacheHit"] = true;
            return context;
        }
        
        // Process and cache result
        context = await next(context);
        
        if (context.Items.TryGetValue("Response", out var response))
        {
            await _cache.SetAsync(
                cacheKey,
                response,
                TimeSpan.FromMinutes(5));
        }
        
        return context;
    }
}
```

</td>
<td>

```go
type CachingInterceptor struct {
    cache        Cache
    keyGenerator CacheKeyGenerator
}

func (i *CachingInterceptor) Intercept(
    ctx context.Context,
    msg contracts.Message,
    next interceptors.Handler) error {
    
    // Only cache queries
    query, isQuery := msg.(contracts.Query)
    if !isQuery {
        return next(ctx, msg)
    }
    
    cacheKey := i.keyGenerator.GenerateKey(query)
    
    // Check cache
    var cached interface{}
    err := i.cache.Get(ctx, cacheKey, &cached)
    if err == nil && cached != nil {
        // Cache hit - store in context
        ctx = context.WithValue(ctx, 
            cachedResponseKey, cached)
        ctx = context.WithValue(ctx,
            cacheHitKey, true)
        return nil
    }
    
    // Process request
    err = next(ctx, msg)
    if err != nil {
        return err
    }
    
    // Cache response if available
    if response := ctx.Value(responseKey); response != nil {
        _ = i.cache.Set(ctx, cacheKey, response,
            5*time.Minute)
    }
    
    return nil
}
```

</td>
</tr>
</table>

## Testing Interceptors

### Unit Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task LoggingInterceptor_LogsBeforeAndAfter()
{
    // Arrange
    var logger = new TestLogger<LoggingInterceptor>();
    var interceptor = new LoggingInterceptor(logger);
    var context = new MessageContext(
        new TestCommand { Id = "123" });
    
    var nextCalled = false;
    MessageDelegate next = async (ctx) =>
    {
        nextCalled = true;
        await Task.Delay(100); // Simulate work
        return ctx;
    };
    
    // Act
    await interceptor.InterceptAsync(context, next);
    
    // Assert
    Assert.IsTrue(nextCalled);
    Assert.AreEqual(2, logger.LogEntries.Count);
    
    var firstLog = logger.LogEntries[0];
    Assert.AreEqual(LogLevel.Information, firstLog.Level);
    Assert.IsTrue(firstLog.Message.Contains("Processing"));
    
    var secondLog = logger.LogEntries[1];
    Assert.AreEqual(LogLevel.Information, secondLog.Level);
    Assert.IsTrue(secondLog.Message.Contains("Processed"));
}
```

</td>
<td>

```go
func TestLoggingInterceptor_LogsBeforeAndAfter(t *testing.T) {
    // Arrange
    var logs []string
    logger := slog.New(slog.NewTextHandler(
        &testWriter{logs: &logs}, nil))
    
    interceptor := &LoggingInterceptor{logger: logger}
    msg := &TestCommand{ID: "123"}
    
    nextCalled := false
    next := interceptors.HandlerFunc(
        func(ctx context.Context, m contracts.Message) error {
            nextCalled = true
            time.Sleep(100 * time.Millisecond)
            return nil
        })
    
    // Act
    err := interceptor.Intercept(
        context.Background(), msg, next)
    
    // Assert
    assert.NoError(t, err)
    assert.True(t, nextCalled)
    assert.Len(t, logs, 2)
    assert.Contains(t, logs[0], "Processing message")
    assert.Contains(t, logs[1], "Processed message")
}
```

</td>
</tr>
</table>

### Integration Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task InterceptorPipeline_ExecutesInOrder()
{
    // Arrange
    var executionOrder = new List<string>();
    
    var interceptor1 = new TestInterceptor("1", executionOrder);
    var interceptor2 = new TestInterceptor("2", executionOrder);
    var interceptor3 = new TestInterceptor("3", executionOrder);
    
    var pipeline = new InterceptorPipeline(
        interceptor1, interceptor2, interceptor3);
    
    var handler = new TestHandler(executionOrder);
    
    // Act
    await pipeline.ExecuteAsync(
        new TestCommand(),
        handler.HandleAsync);
    
    // Assert
    CollectionAssert.AreEqual(
        new[] { "1-pre", "2-pre", "3-pre", "handler", 
                "3-post", "2-post", "1-post" },
        executionOrder);
}
```

</td>
<td>

```go
func TestInterceptorPipeline_ExecutesInOrder(t *testing.T) {
    // Arrange
    var executionOrder []string
    var mu sync.Mutex
    
    recordExecution := func(event string) {
        mu.Lock()
        executionOrder = append(executionOrder, event)
        mu.Unlock()
    }
    
    interceptor1 := &TestInterceptor{
        name: "1", record: recordExecution}
    interceptor2 := &TestInterceptor{
        name: "2", record: recordExecution}
    interceptor3 := &TestInterceptor{
        name: "3", record: recordExecution}
    
    pipeline := interceptors.NewPipeline(
        interceptor1, interceptor2, interceptor3)
    
    handler := interceptors.HandlerFunc(
        func(ctx context.Context, msg contracts.Message) error {
            recordExecution("handler")
            return nil
        })
    
    // Act
    err := pipeline.Execute(
        context.Background(),
        &TestCommand{},
        handler)
    
    // Assert
    assert.NoError(t, err)
    assert.Equal(t, []string{
        "1-pre", "2-pre", "3-pre", "handler",
        "3-post", "2-post", "1-post",
    }, executionOrder)
}
```

</td>
</tr>
</table>

## Performance Considerations

### Lightweight Interceptors

Keep interceptors fast:
- Avoid blocking I/O in the main path
- Use async operations where possible
- Cache expensive computations
- Consider sampling for metrics

### Order Optimization

Order interceptors by:
1. Security/Authorization (fail fast)
2. Validation (fail fast)
3. Caching (skip processing)
4. Metrics/Logging (always run)
5. Business logic interceptors

### Conditional Execution

Skip expensive interceptors when not needed:
- Check message type
- Check headers/metadata
- Use sampling for metrics
- Cache authorization results

## Best Practices

1. **Single Responsibility**
   - Each interceptor should do one thing well
   - Compose multiple interceptors for complex behavior
   - Keep interceptors stateless when possible

2. **Error Handling**
   - Don't swallow exceptions
   - Add context to errors
   - Log errors appropriately
   - Consider retry strategies

3. **Performance**
   - Measure interceptor overhead
   - Use async operations
   - Implement timeouts
   - Cache when appropriate

4. **Testing**
   - Unit test each interceptor
   - Test interceptor ordering
   - Test error scenarios
   - Mock external dependencies

5. **Configuration**
   - Make interceptors configurable
   - Support enabling/disabling
   - Allow runtime changes
   - Document configuration options

## Common Patterns

### Request/Response Correlation
Track related messages across services

### Idempotency
Ensure operations can be safely retried

### Rate Limiting
Protect services from overload

### Audit Trail
Track all message processing for compliance

### Message Enrichment
Add context data to messages

## Next Steps

- Review [Examples](../../README.md#examples) for real-world usage
- Learn about [Performance Tuning](../../advanced/performance.md)
- Implement [Monitoring](../monitoring/README.md)
- Study [Security Patterns](../../advanced/security.md)