# Security

This guide covers security implementation patterns and best practices for Mmate applications.

## Authentication

### Token-Based Authentication

Implement authentication using message interceptors:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class AuthenticationInterceptor : 
    MessageInterceptorBase
{
    private readonly ITokenValidator _tokenValidator;
    
    public override async Task OnBeforeConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        // Extract token from headers
        var token = context.GetHeader<string>("Authorization");
        if (string.IsNullOrEmpty(token))
        {
            throw new UnauthorizedException(
                "Authentication token required");
        }
        
        // Validate token
        var principal = await _tokenValidator
            .ValidateAsync(token);
        if (principal == null)
        {
            throw new UnauthorizedException(
                "Invalid authentication token");
        }
        
        // Store authenticated user in context
        context.Properties["User"] = principal;
        context.Properties["TenantId"] = 
            principal.GetTenantId();
    }
}
```

</td>
<td>

```go
type AuthenticationInterceptor struct {
    tokenValidator TokenValidator
}

func (i *AuthenticationInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) (context.Context, error) {
    
    // Extract token from headers
    token := msg.Headers()["Authorization"]
    if token == "" {
        return nil, errors.New(
            "authentication token required")
    }
    
    // Validate token
    principal, err := i.tokenValidator.Validate(
        ctx, token)
    if err != nil {
        return nil, fmt.Errorf(
            "invalid token: %w", err)
    }
    
    // Store authenticated user in context
    ctx = context.WithValue(ctx, "user", principal)
    ctx = context.WithValue(ctx, "tenantId", 
        principal.TenantID)
    
    return ctx, nil
}
```

</td>
</tr>
</table>

## Authorization

### Permission-Based Access Control

Control access to specific message types:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class AuthorizationInterceptor : 
    MessageInterceptorBase
{
    public override Task OnBeforeConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        var user = context.Properties["User"] 
            as ClaimsPrincipal;
        var messageType = context.Message.GetType();
        
        // Check permissions based on message type
        var requiredPermission = GetRequiredPermission(
            messageType);
        
        if (!user.HasPermission(requiredPermission))
        {
            throw new ForbiddenException(
                $"Access denied for {messageType.Name}");
        }
        
        // Resource-level authorization
        if (context.Message is IResourceMessage resource)
        {
            if (!user.CanAccessResource(resource.ResourceId))
            {
                throw new ForbiddenException(
                    "Access denied to resource");
            }
        }
        
        return Task.CompletedTask;
    }
}

// Message attribute for permissions
[RequiresPermission("orders.create", "payment.process")]
public class CreateOrderCommand : BaseCommand, 
    IResourceMessage
{
    public string ResourceId => CustomerId;
    public string CustomerId { get; set; }
}
```

</td>
<td>

```go
type AuthorizationInterceptor struct {
    permissions PermissionService
}

func (i *AuthorizationInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) (context.Context, error) {
    
    user := ctx.Value("user").(Principal)
    
    // Check permissions based on message type
    required := i.getRequiredPermission(msg.Type())
    if !user.HasPermission(required) {
        return nil, fmt.Errorf(
            "access denied for %s", msg.Type())
    }
    
    // Resource-level authorization
    if resourceMsg, ok := msg.(ResourceMessage); ok {
        if !user.CanAccessResource(
            resourceMsg.ResourceID()) {
            return nil, errors.New(
                "access denied to resource")
        }
    }
    
    return ctx, nil
}

// Message interface for resource access
type ResourceMessage interface {
    ResourceID() string
}

type CreateOrderCommand struct {
    CustomerID string `json:"customerId"`
}

func (c *CreateOrderCommand) ResourceID() string {
    return c.CustomerID
}
```

</td>
</tr>
</table>

## Multi-Tenant Security

### Tenant Isolation

Ensure data isolation between tenants:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class TenantIsolationInterceptor : 
    MessageInterceptorBase
{
    public override Task OnBeforeConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        var tenantId = context.Properties["TenantId"] 
            as string;
        
        // Set tenant context for database queries
        _tenantContext.SetCurrentTenant(tenantId);
        
        // Validate message belongs to tenant
        if (context.Message is ITenantMessage tenantMsg)
        {
            if (tenantMsg.TenantId != tenantId)
            {
                throw new ForbiddenException(
                    "Access denied to tenant data");
            }
        }
        
        return Task.CompletedTask;
    }
}
```

</td>
<td>

```go
type TenantIsolationInterceptor struct {
    tenantContext TenantContext
}

func (i *TenantIsolationInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) (context.Context, error) {
    
    tenantID := ctx.Value("tenantId").(string)
    
    // Set tenant context for database queries
    ctx = i.tenantContext.WithTenant(ctx, tenantID)
    
    // Validate message belongs to tenant
    if tenantMsg, ok := msg.(TenantMessage); ok {
        if tenantMsg.TenantID() != tenantID {
            return nil, errors.New(
                "access denied to tenant data")
        }
    }
    
    return ctx, nil
}
```

</td>
</tr>
</table>

## Transport Security

### TLS/SSL Configuration

Secure transport layer communication:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateMessaging()
    .WithRabbitMqTransport(options =>
    {
        // Use TLS connection
        options.ConnectionString = 
            "amqps://user:pass@broker.com:5671/";
        
        // Client certificate authentication
        options.EnableTls = true;
        options.ClientCertificatePath = 
            configuration["Security:ClientCertPath"];
        options.ClientCertificatePassword = 
            configuration["Security:ClientCertPassword"];
        
        // Certificate validation
        options.ValidateServerCertificate = true;
        options.ServerCertificateName = "broker.company.com";
    });
```

</td>
<td>

```go
// TLS configuration
tlsConfig := &tls.Config{
    ClientCAs: clientCAs,
    Certificates: []tls.Certificate{clientCert},
    ServerName: "broker.company.com",
    MinVersion: tls.VersionTLS12,
}

// RabbitMQ with TLS
config := rabbitmq.Config{
    URL: "amqps://user:pass@broker.com:5671/",
    TLS: tlsConfig,
}
```

</td>
</tr>
</table>

## Message Encryption

### End-to-End Message Encryption

Encrypt sensitive message content:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class EncryptionInterceptor : 
    MessageInterceptorBase
{
    private readonly IEncryptionService _encryption;
    
    public override async Task OnBeforePublishAsync(
        MessageContext context,
        CancellationToken ct)
    {
        // Encrypt sensitive messages
        if (context.Message is ISensitiveMessage)
        {
            var json = JsonSerializer.Serialize(
                context.Message);
            var encrypted = await _encryption
                .EncryptAsync(json);
            
            // Replace with encrypted envelope
            context.Message = new EncryptedMessage
            {
                EncryptedPayload = encrypted,
                MessageType = context.Message.GetType()
                    .AssemblyQualifiedName
            };
        }
    }
    
    public override async Task OnAfterConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        // Decrypt if needed
        if (context.Message is EncryptedMessage encrypted)
        {
            var decrypted = await _encryption
                .DecryptAsync(encrypted.EncryptedPayload);
            var messageType = Type.GetType(
                encrypted.MessageType);
            
            context.Message = JsonSerializer
                .Deserialize(decrypted, messageType);
        }
    }
}
```

</td>
<td>

```go
type EncryptionInterceptor struct {
    encryption EncryptionService
}

func (i *EncryptionInterceptor) BeforePublish(
    ctx context.Context,
    msg messaging.Message) (messaging.Message, error) {
    
    // Encrypt sensitive messages
    if _, ok := msg.(SensitiveMessage); ok {
        data, err := json.Marshal(msg)
        if err != nil {
            return nil, err
        }
        
        encrypted, err := i.encryption.Encrypt(ctx, data)
        if err != nil {
            return nil, err
        }
        
        // Replace with encrypted envelope
        return &EncryptedMessage{
            EncryptedPayload: encrypted,
            MessageType:      msg.Type(),
        }, nil
    }
    
    return msg, nil
}

func (i *EncryptionInterceptor) AfterConsume(
    ctx context.Context,
    msg messaging.Message) (messaging.Message, error) {
    
    // Decrypt if needed
    if encMsg, ok := msg.(*EncryptedMessage); ok {
        decrypted, err := i.encryption.Decrypt(
            ctx, encMsg.EncryptedPayload)
        if err != nil {
            return nil, err
        }
        
        return i.deserializeMessage(
            encMsg.MessageType, decrypted)
    }
    
    return msg, nil
}
```

</td>
</tr>
</table>

## Rate Limiting and Protection

### Anti-Abuse Measures

Protect against message abuse:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class RateLimitingInterceptor : 
    MessageInterceptorBase
{
    private readonly IRateLimiter _rateLimiter;
    
    public override async Task OnBeforeConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        var user = context.Properties["User"] 
            as ClaimsPrincipal;
        var tenantId = context.Properties["TenantId"] 
            as string;
        
        // User-level rate limiting
        var userKey = $"user:{user.GetUserId()}";
        if (!await _rateLimiter.TryAcquireAsync(
            userKey, 100, TimeSpan.FromMinutes(1)))
        {
            throw new RateLimitExceededException(
                "User rate limit exceeded");
        }
        
        // Tenant-level rate limiting
        var tenantKey = $"tenant:{tenantId}";
        if (!await _rateLimiter.TryAcquireAsync(
            tenantKey, 1000, TimeSpan.FromMinutes(1)))
        {
            throw new RateLimitExceededException(
                "Tenant rate limit exceeded");
        }
    }
}
```

</td>
<td>

```go
type RateLimitingInterceptor struct {
    rateLimiter RateLimiter
}

func (i *RateLimitingInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) (context.Context, error) {
    
    user := ctx.Value("user").(Principal)
    tenantID := ctx.Value("tenantId").(string)
    
    // User-level rate limiting
    userKey := fmt.Sprintf("user:%s", user.ID)
    if !i.rateLimiter.TryAcquire(
        userKey, 100, time.Minute) {
        return nil, errors.New(
            "user rate limit exceeded")
    }
    
    // Tenant-level rate limiting
    tenantKey := fmt.Sprintf("tenant:%s", tenantID)
    if !i.rateLimiter.TryAcquire(
        tenantKey, 1000, time.Minute) {
        return nil, errors.New(
            "tenant rate limit exceeded")
    }
    
    return ctx, nil
}
```

</td>
</tr>
</table>

## Compliance and Auditing

### HIPAA Compliance Example

For healthcare applications:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class HIPAAComplianceInterceptor : 
    MessageInterceptorBase
{
    public override async Task OnBeforeConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        if (context.Message is IContainsPHI phiMessage)
        {
            // Verify patient consent
            await VerifyPatientConsent(phiMessage.PatientId);
            
            // Decrypt PHI data
            phiMessage.DecryptPHI(_encryptionKey);
            
            // Audit access
            await _auditLogger.LogPHIAccess(
                phiMessage.PatientId,
                context.Properties["User"] as ClaimsPrincipal,
                context.Message.GetType().Name);
        }
    }
    
    public override async Task OnAfterConsumeAsync(
        MessageContext context,
        CancellationToken ct)
    {
        if (context.Message is IContainsPHI phiMessage)
        {
            // Re-encrypt PHI data
            phiMessage.EncryptPHI(_encryptionKey);
        }
    }
}
```

</td>
<td>

```go
type HIPAAComplianceInterceptor struct {
    encryptionKey []byte
    auditLogger   AuditLogger
}

func (i *HIPAAComplianceInterceptor) Before(
    ctx context.Context,
    msg messaging.Message) (context.Context, error) {
    
    if phiMsg, ok := msg.(PHIMessage); ok {
        // Verify patient consent
        if err := i.verifyPatientConsent(
            phiMsg.PatientID()); err != nil {
            return nil, err
        }
        
        // Decrypt PHI data
        if err := phiMsg.DecryptPHI(
            i.encryptionKey); err != nil {
            return nil, err
        }
        
        // Audit access
        user := ctx.Value("user").(Principal)
        i.auditLogger.LogPHIAccess(
            phiMsg.PatientID(), user, msg.Type())
    }
    
    return ctx, nil
}
```

</td>
</tr>
</table>

## Security Configuration

### Security Interceptor Chain

Configure security layers in the correct order:

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateInterceptors(interceptors =>
{
    // Order matters!
    interceptors.Add<EncryptionInterceptor>(order: 1);
    interceptors.Add<AuthenticationInterceptor>(order: 2);
    interceptors.Add<AuthorizationInterceptor>(order: 3);
    interceptors.Add<TenantIsolationInterceptor>(order: 4);
    interceptors.Add<RateLimitingInterceptor>(order: 5);
    interceptors.Add<AuditInterceptor>(order: 6);
});
```

</td>
<td>

```go
// Order matters!
interceptors := []messaging.Interceptor{
    &EncryptionInterceptor{},
    &AuthenticationInterceptor{},
    &AuthorizationInterceptor{},
    &TenantIsolationInterceptor{},
    &RateLimitingInterceptor{},
    &AuditInterceptor{},
}

dispatcher := messaging.NewDispatcher(
    messaging.WithInterceptors(interceptors...))
```

</td>
</tr>
</table>

## Best Practices

### Security Checklist

1. **Transport Security**
   - [ ] Use TLS for all connections
   - [ ] Validate server certificates
   - [ ] Use strong cipher suites

2. **Authentication**
   - [ ] Require authentication for all messages
   - [ ] Use strong token validation
   - [ ] Implement token expiration

3. **Authorization**
   - [ ] Implement fine-grained permissions
   - [ ] Use resource-level access control
   - [ ] Validate message ownership

4. **Data Protection**
   - [ ] Encrypt sensitive message content
   - [ ] Use strong encryption algorithms
   - [ ] Protect encryption keys

5. **Monitoring**
   - [ ] Log all security events
   - [ ] Monitor for abuse patterns
   - [ ] Set up security alerts

6. **Compliance**
   - [ ] Implement required compliance controls
   - [ ] Audit access to sensitive data
   - [ ] Document security procedures

### Common Security Patterns

1. **Defense in Depth**: Layer multiple security controls
2. **Principle of Least Privilege**: Grant minimal required permissions
3. **Zero Trust**: Verify every request regardless of source
4. **Audit Everything**: Log security-relevant events
5. **Fail Secure**: Default to denying access on errors