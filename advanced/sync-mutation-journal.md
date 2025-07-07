# Sync Mutation Journal

The sync mutation journal provides entity-level mutation tracking for distributed system synchronization. It records all entity changes with detailed metadata, sync status tracking, and conflict resolution capabilities, enabling reliable data synchronization across services and environments.

## How It Works

The sync mutation journal extends the basic mutation journal with synchronization-specific features:

1. **Entity-Level Tracking**: Records mutations at the entity level (create, update, delete, patch, merge)
2. **Sync Status Management**: Tracks sync status (pending, synced, failed, conflict, ignored)
3. **Correlation Tracking**: Groups related mutations using correlation IDs
4. **Conflict Detection**: Identifies and manages sync conflicts between services
5. **Before/After State**: Captures entity state before and after mutations
6. **Metadata Enrichment**: Stores additional context for synchronization decisions

## Key Benefits

- **Distributed Synchronization**: Reliable entity sync across multiple services
- **Conflict Resolution**: Automatic detection and manual resolution of sync conflicts  
- **Audit Trail**: Complete history of entity mutations and sync status
- **Recovery Support**: Replay failed mutations or resolve conflicts
- **Performance Tracking**: Monitor sync performance and identify bottlenecks

## Basic Configuration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
services.AddMmateMessaging()
    .WithSyncMutationJournal(options =>
    {
        options.ServiceId = "order-service";
        options.MaxEntries = 50000;
        options.RetentionPeriod = TimeSpan.FromDays(30);
        options.EnableConflictDetection = true;
        options.SyncQueueName = "mmate.sync.mutations";
    });
```

</td>
<td>

```go
import (
    "github.com/glimte/mmate-go"
    "github.com/glimte/mmate-go/internal/journal"
)

// Create client with sync mutation journal enabled
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithSyncMutationJournal(
        journal.WithServiceID("order-service"),
    ),
    mmate.WithServiceName("order-service"),
)
if err != nil {
    log.Fatal("Failed to create client:", err)
}
```

</td>
</tr>
</table>

## Recording Entity Mutations

### Basic Entity Mutations

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class OrderService
{
    private readonly ISyncMutationJournal _journal;
    
    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        var order = new Order
        {
            Id = command.OrderId,
            CustomerId = command.CustomerId,
            Status = OrderStatus.Created,
            CreatedAt = DateTime.UtcNow
        };
        
        // Save order to database
        await _repository.SaveAsync(order);
        
        // Record entity mutation
        var mutation = new EntityMutationRecord
        {
            EntityType = "Order",
            EntityId = order.Id,
            EntityVersion = 1,
            MutationType = EntityMutationType.Create,
            CorrelationId = command.CorrelationId,
            AfterState = JsonSerializer.Serialize(order),
            SyncStatus = SyncStatus.Pending,
            Metadata = new Dictionary<string, object>
            {
                ["command"] = command.GetType().Name,
                ["userId"] = command.UserId,
                ["source"] = "api"
            }
        };
        
        await _journal.RecordEntityMutationAsync(mutation);
        return order;
    }
}
```

</td>
<td>

```go
func (s *OrderService) CreateOrder(
    ctx context.Context, 
    command *CreateOrderCommand) (*Order, error) {
    
    order := &Order{
        ID:         command.OrderID,
        CustomerID: command.CustomerID,
        Status:     OrderStatusCreated,
        CreatedAt:  time.Now(),
    }
    
    // Save order to database
    err := s.repository.Save(ctx, order)
    if err != nil {
        return nil, fmt.Errorf("failed to save order: %w", err)
    }
    
    // Serialize after state
    afterState, err := json.Marshal(order)
    if err != nil {
        return nil, fmt.Errorf("failed to serialize order: %w", err)
    }
    
    // Record entity mutation
    mutation := &journal.EntityMutationRecord{
        EntityType:    "Order",
        EntityID:      order.ID,
        EntityVersion: 1,
        MutationType:  journal.EntityCreate,
        CorrelationID: command.GetCorrelationID(),
        AfterState:    json.RawMessage(afterState),
        SyncStatus:    journal.SyncStatusPending,
        Metadata: map[string]interface{}{
            "command": "CreateOrderCommand",
            "userId":  command.UserID,
            "source":  "api",
        },
    }
    
    err = s.client.RecordEntityMutation(ctx, mutation)
    if err != nil {
        log.Printf("Failed to record mutation: %v", err)
        // Don't fail the operation, just log the error
    }
    
    return order, nil
}
```

</td>
</tr>
</table>

### Update Mutations with Before/After State

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task<Order> UpdateOrderStatusAsync(
    string orderId, 
    OrderStatus newStatus,
    string correlationId)
{
    // Get current state
    var order = await _repository.GetByIdAsync(orderId);
    var beforeState = JsonSerializer.Serialize(order);
    
    // Update order
    order.Status = newStatus;
    order.UpdatedAt = DateTime.UtcNow;
    order.Version++;
    
    await _repository.SaveAsync(order);
    
    // Record mutation with before/after state
    var mutation = new EntityMutationRecord
    {
        EntityType = "Order",
        EntityId = order.Id,
        EntityVersion = order.Version,
        MutationType = EntityMutationType.Update,
        CorrelationId = correlationId,
        BeforeState = beforeState,
        AfterState = JsonSerializer.Serialize(order),
        SyncStatus = SyncStatus.Pending,
        Metadata = new Dictionary<string, object>
        {
            ["field"] = "Status",
            ["oldValue"] = order.Status.ToString(),
            ["newValue"] = newStatus.ToString()
        }
    };
    
    await _journal.RecordEntityMutationAsync(mutation);
    return order;
}
```

</td>
<td>

```go
func (s *OrderService) UpdateOrderStatus(
    ctx context.Context, 
    orderID string, 
    newStatus OrderStatus,
    correlationID string) (*Order, error) {
    
    // Get current state
    order, err := s.repository.GetByID(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("failed to get order: %w", err)
    }
    
    // Serialize before state
    beforeState, err := json.Marshal(order)
    if err != nil {
        return nil, fmt.Errorf("failed to serialize before state: %w", err)
    }
    
    oldStatus := order.Status
    
    // Update order
    order.Status = newStatus
    order.UpdatedAt = time.Now()
    order.Version++
    
    err = s.repository.Save(ctx, order)
    if err != nil {
        return nil, fmt.Errorf("failed to save order: %w", err)
    }
    
    // Serialize after state
    afterState, err := json.Marshal(order)
    if err != nil {
        return nil, fmt.Errorf("failed to serialize after state: %w", err)
    }
    
    // Record mutation with before/after state
    mutation := &journal.EntityMutationRecord{
        EntityType:    "Order",
        EntityID:      order.ID,
        EntityVersion: order.Version,
        MutationType:  journal.EntityUpdate,
        CorrelationID: correlationID,
        BeforeState:   json.RawMessage(beforeState),
        AfterState:    json.RawMessage(afterState),
        SyncStatus:    journal.SyncStatusPending,
        Metadata: map[string]interface{}{
            "field":    "Status",
            "oldValue": oldStatus.String(),
            "newValue": newStatus.String(),
        },
    }
    
    err = s.client.RecordEntityMutation(ctx, mutation)
    if err != nil {
        log.Printf("Failed to record mutation: %v", err)
    }
    
    return order, nil
}
```

</td>
</tr>
</table>

## Querying Mutations

### Get Entity Mutations

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task<List<EntityMutationRecord>> GetOrderHistoryAsync(string orderId)
{
    // Get all mutations for a specific order
    var mutations = await _journal.GetEntityMutationsAsync("Order", orderId);
    
    // Order by timestamp to show chronological history
    return mutations.OrderBy(m => m.Timestamp).ToList();
}

public async Task<OrderSyncStatus> GetOrderSyncStatusAsync(string orderId)
{
    var mutations = await _journal.GetEntityMutationsAsync("Order", orderId);
    var latestMutation = mutations.OrderByDescending(m => m.Timestamp).FirstOrDefault();
    
    return new OrderSyncStatus
    {
        OrderId = orderId,
        SyncStatus = latestMutation?.SyncStatus ?? SyncStatus.Unknown,
        LastSyncedAt = latestMutation?.SyncedAt,
        LastError = latestMutation?.SyncError,
        Version = latestMutation?.EntityVersion ?? 0
    };
}
```

</td>
<td>

```go
func (s *OrderService) GetOrderHistory(
    ctx context.Context, 
    orderID string) ([]*journal.EntityMutationRecord, error) {
    
    // Get all mutations for a specific order
    mutations, err := s.client.GetEntityMutations(ctx, "Order", orderID)
    if err != nil {
        return nil, fmt.Errorf("failed to get entity mutations: %w", err)
    }
    
    // Sort by timestamp for chronological history
    sort.Slice(mutations, func(i, j int) bool {
        return mutations[i].Timestamp.Before(mutations[j].Timestamp)
    })
    
    return mutations, nil
}

type OrderSyncStatus struct {
    OrderID      string                 `json:"orderId"`
    SyncStatus   journal.SyncStatus     `json:"syncStatus"`
    LastSyncedAt *time.Time             `json:"lastSyncedAt,omitempty"`
    LastError    string                 `json:"lastError,omitempty"`
    Version      int64                  `json:"version"`
}

func (s *OrderService) GetOrderSyncStatus(
    ctx context.Context, 
    orderID string) (*OrderSyncStatus, error) {
    
    mutations, err := s.client.GetEntityMutations(ctx, "Order", orderID)
    if err != nil {
        return nil, fmt.Errorf("failed to get entity mutations: %w", err)
    }
    
    if len(mutations) == 0 {
        return &OrderSyncStatus{
            OrderID:    orderID,
            SyncStatus: journal.SyncStatus("unknown"),
        }, nil
    }
    
    // Get latest mutation
    latestMutation := mutations[len(mutations)-1]
    
    return &OrderSyncStatus{
        OrderID:      orderID,
        SyncStatus:   latestMutation.SyncStatus,
        LastSyncedAt: latestMutation.SyncedAt,
        LastError:    latestMutation.SyncError,
        Version:      latestMutation.EntityVersion,
    }, nil
}
```

</td>
</tr>
</table>

### Query Unsynced Mutations

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class SyncService
{
    private readonly ISyncMutationJournal _journal;
    private readonly IExternalSyncClient _externalSync;
    
    public async Task SyncPendingMutationsAsync()
    {
        // Get mutations that need to be synced
        var unsyncedMutations = await _journal.GetUnsyncedMutationsAsync(limit: 100);
        
        foreach (var mutation in unsyncedMutations)
        {
            try
            {
                // Sync with external system
                await _externalSync.SyncEntityAsync(
                    mutation.EntityType,
                    mutation.EntityId,
                    mutation.AfterState,
                    mutation.EntityVersion);
                
                // Mark as synced
                await _journal.MarkAsSyncedAsync(
                    new[] { mutation.Id }, 
                    DateTime.UtcNow);
                
                _logger.LogInformation("Synced {EntityType} {EntityId} version {Version}",
                    mutation.EntityType, mutation.EntityId, mutation.EntityVersion);
            }
            catch (Exception ex)
            {
                // Mark as failed
                await _journal.MarkAsFailedAsync(
                    new[] { mutation.Id }, 
                    ex.Message);
                
                _logger.LogError(ex, "Failed to sync {EntityType} {EntityId}",
                    mutation.EntityType, mutation.EntityId);
            }
        }
    }
}
```

</td>
<td>

```go
type SyncService struct {
    client       *mmate.Client
    externalSync ExternalSyncClient
    logger       *slog.Logger
}

func (s *SyncService) SyncPendingMutations(ctx context.Context) error {
    // Get mutations that need to be synced
    unsyncedMutations, err := s.client.GetUnsyncedMutations(ctx, 100)
    if err != nil {
        return fmt.Errorf("failed to get unsynced mutations: %w", err)
    }
    
    for _, mutation := range unsyncedMutations {
        err := s.syncMutation(ctx, mutation)
        if err != nil {
            s.logger.Error("Failed to sync mutation", 
                "mutationId", mutation.ID,
                "entityType", mutation.EntityType,
                "entityId", mutation.EntityID,
                "error", err)
        }
    }
    
    return nil
}

func (s *SyncService) syncMutation(
    ctx context.Context, 
    mutation *journal.EntityMutationRecord) error {
    
    // Sync with external system
    err := s.externalSync.SyncEntity(
        ctx,
        mutation.EntityType,
        mutation.EntityID,
        mutation.AfterState,
        mutation.EntityVersion)
    
    if err != nil {
        // Mark as failed
        journal := s.client.SyncMutationJournal()
        markErr := journal.MarkAsFailed(
            ctx, 
            []string{mutation.ID}, 
            err.Error())
        if markErr != nil {
            s.logger.Error("Failed to mark mutation as failed", 
                "mutationId", mutation.ID, "error", markErr)
        }
        return fmt.Errorf("external sync failed: %w", err)
    }
    
    // Mark as synced
    journal := s.client.SyncMutationJournal()
    err = journal.MarkAsSynced(
        ctx, 
        []string{mutation.ID}, 
        time.Now())
    if err != nil {
        s.logger.Error("Failed to mark mutation as synced", 
            "mutationId", mutation.ID, "error", err)
        return fmt.Errorf("failed to mark as synced: %w", err)
    }
    
    s.logger.Info("Synced entity mutation",
        "entityType", mutation.EntityType,
        "entityId", mutation.EntityID,
        "version", mutation.EntityVersion)
    
    return nil
}
```

</td>
</tr>
</table>

## Correlation-Based Queries

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public async Task<List<EntityMutationRecord>> GetTransactionMutationsAsync(
    string correlationId)
{
    // Get all mutations that are part of the same transaction/operation
    var mutations = await _journal.GetCorrelatedMutationsAsync(correlationId);
    
    // Group by entity for analysis
    var mutationsByEntity = mutations
        .GroupBy(m => new { m.EntityType, m.EntityId })
        .ToDictionary(
            g => g.Key, 
            g => g.OrderBy(m => m.Timestamp).ToList());
    
    return mutations;
}

public async Task<TransactionSyncStatus> GetTransactionSyncStatusAsync(
    string correlationId)
{
    var mutations = await _journal.GetCorrelatedMutationsAsync(correlationId);
    
    var totalMutations = mutations.Count;
    var syncedMutations = mutations.Count(m => m.SyncStatus == SyncStatus.Synced);
    var failedMutations = mutations.Count(m => m.SyncStatus == SyncStatus.Failed);
    var pendingMutations = mutations.Count(m => m.SyncStatus == SyncStatus.Pending);
    
    return new TransactionSyncStatus
    {
        CorrelationId = correlationId,
        TotalMutations = totalMutations,
        SyncedMutations = syncedMutations,
        FailedMutations = failedMutations,
        PendingMutations = pendingMutations,
        IsComplete = pendingMutations == 0 && failedMutations == 0,
        SuccessRate = totalMutations > 0 ? (double)syncedMutations / totalMutations : 1.0
    };
}
```

</td>
<td>

```go
func (s *SyncService) GetTransactionMutations(
    ctx context.Context, 
    correlationID string) ([]*journal.EntityMutationRecord, error) {
    
    // Get all mutations that are part of the same transaction/operation
    journal := s.client.SyncMutationJournal()
    mutations, err := journal.GetCorrelatedMutations(ctx, correlationID)
    if err != nil {
        return nil, fmt.Errorf("failed to get correlated mutations: %w", err)
    }
    
    // Sort by timestamp for chronological order
    sort.Slice(mutations, func(i, j int) bool {
        return mutations[i].Timestamp.Before(mutations[j].Timestamp)
    })
    
    return mutations, nil
}

type TransactionSyncStatus struct {
    CorrelationID    string  `json:"correlationId"`
    TotalMutations   int     `json:"totalMutations"`
    SyncedMutations  int     `json:"syncedMutations"`
    FailedMutations  int     `json:"failedMutations"`
    PendingMutations int     `json:"pendingMutations"`
    IsComplete       bool    `json:"isComplete"`
    SuccessRate      float64 `json:"successRate"`
}

func (s *SyncService) GetTransactionSyncStatus(
    ctx context.Context, 
    correlationID string) (*TransactionSyncStatus, error) {
    
    mutations, err := s.GetTransactionMutations(ctx, correlationID)
    if err != nil {
        return nil, err
    }
    
    status := &TransactionSyncStatus{
        CorrelationID:  correlationID,
        TotalMutations: len(mutations),
    }
    
    for _, mutation := range mutations {
        switch mutation.SyncStatus {
        case journal.SyncStatusSynced:
            status.SyncedMutations++
        case journal.SyncStatusFailed:
            status.FailedMutations++
        case journal.SyncStatusPending:
            status.PendingMutations++
        }
    }
    
    status.IsComplete = status.PendingMutations == 0 && status.FailedMutations == 0
    if status.TotalMutations > 0 {
        status.SuccessRate = float64(status.SyncedMutations) / float64(status.TotalMutations)
    } else {
        status.SuccessRate = 1.0
    }
    
    return status, nil
}
```

</td>
</tr>
</table>

## Conflict Detection and Resolution

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ConflictResolutionService
{
    private readonly ISyncMutationJournal _journal;
    
    public async Task<List<EntityMutationRecord>> GetConflictsAsync()
    {
        return await _journal.GetConflictsAsync();
    }
    
    public async Task ResolveConflictAsync(
        string mutationId, 
        ConflictResolution resolution)
    {
        var newStatus = resolution switch
        {
            ConflictResolution.AcceptLocal => SyncStatus.Synced,
            ConflictResolution.AcceptRemote => SyncStatus.Ignored,
            ConflictResolution.Merge => SyncStatus.Pending, // Retry with merged state
            ConflictResolution.ManualReview => SyncStatus.Conflict,
            _ => throw new ArgumentException("Invalid resolution")
        };
        
        await _journal.ResolveConflictAsync(mutationId, newStatus);
        
        _logger.LogInformation("Resolved conflict for mutation {MutationId} with {Resolution}",
            mutationId, resolution);
    }
    
    public async Task HandleIncomingMutationAsync(
        string entityType,
        string entityId,
        long remoteVersion,
        object remoteState)
    {
        // Check for conflicts with local mutations
        var localMutations = await _journal.GetEntityMutationsAsync(entityType, entityId);
        var latestLocal = localMutations.OrderByDescending(m => m.EntityVersion).FirstOrDefault();
        
        if (latestLocal != null && latestLocal.EntityVersion >= remoteVersion)
        {
            // Conflict detected - mark local mutation as conflicted
            await _journal.MarkAsConflictAsync(
                latestLocal.Id,
                $"Conflict with remote version {remoteVersion}");
            
            _logger.LogWarning("Sync conflict detected for {EntityType} {EntityId}: " +
                "local version {LocalVersion}, remote version {RemoteVersion}",
                entityType, entityId, latestLocal.EntityVersion, remoteVersion);
        }
    }
}

public enum ConflictResolution
{
    AcceptLocal,
    AcceptRemote,
    Merge,
    ManualReview
}
```

</td>
<td>

```go
type ConflictResolutionService struct {
    client *mmate.Client
    logger *slog.Logger
}

func (s *ConflictResolutionService) GetConflicts(
    ctx context.Context) ([]*journal.EntityMutationRecord, error) {
    
    journal := s.client.SyncMutationJournal()
    return journal.GetConflicts(ctx)
}

type ConflictResolution int

const (
    ConflictResolutionAcceptLocal ConflictResolution = iota
    ConflictResolutionAcceptRemote
    ConflictResolutionMerge
    ConflictResolutionManualReview
)

func (s *ConflictResolutionService) ResolveConflict(
    ctx context.Context, 
    mutationID string, 
    resolution ConflictResolution) error {
    
    var newStatus journal.SyncStatus
    switch resolution {
    case ConflictResolutionAcceptLocal:
        newStatus = journal.SyncStatusSynced
    case ConflictResolutionAcceptRemote:
        newStatus = journal.SyncStatusIgnored
    case ConflictResolutionMerge:
        newStatus = journal.SyncStatusPending // Retry with merged state
    case ConflictResolutionManualReview:
        newStatus = journal.SyncStatusConflict
    default:
        return fmt.Errorf("invalid conflict resolution: %d", resolution)
    }
    
    journal := s.client.SyncMutationJournal()
    err := journal.ResolveConflict(ctx, mutationID, newStatus)
    if err != nil {
        return fmt.Errorf("failed to resolve conflict: %w", err)
    }
    
    s.logger.Info("Resolved conflict", 
        "mutationId", mutationID, 
        "resolution", resolution)
    
    return nil
}

func (s *ConflictResolutionService) HandleIncomingMutation(
    ctx context.Context,
    entityType, entityID string,
    remoteVersion int64,
    remoteState json.RawMessage) error {
    
    // Check for conflicts with local mutations
    localMutations, err := s.client.GetEntityMutations(ctx, entityType, entityID)
    if err != nil {
        return fmt.Errorf("failed to get local mutations: %w", err)
    }
    
    if len(localMutations) == 0 {
        return nil // No local mutations, no conflict
    }
    
    // Find latest local mutation
    latestLocal := localMutations[len(localMutations)-1]
    
    if latestLocal.EntityVersion >= remoteVersion {
        // Conflict detected - create conflict record
        journal := s.client.SyncMutationJournal()
        
        conflictMutation := &journal.EntityMutationRecord{
            EntityType:    entityType,
            EntityID:      entityID,
            EntityVersion: remoteVersion,
            MutationType:  journal.EntityUpdate,
            SyncStatus:    journal.SyncStatusConflict,
            AfterState:    remoteState,
            ConflictsWith: []string{latestLocal.ID},
            Metadata: map[string]interface{}{
                "conflictType":     "version",
                "localVersion":     latestLocal.EntityVersion,
                "remoteVersion":    remoteVersion,
                "detectedAt":       time.Now(),
            },
        }
        
        err := journal.RecordEntityMutation(ctx, conflictMutation)
        if err != nil {
            return fmt.Errorf("failed to record conflict: %w", err)
        }
        
        s.logger.Warn("Sync conflict detected",
            "entityType", entityType,
            "entityId", entityID,
            "localVersion", latestLocal.EntityVersion,
            "remoteVersion", remoteVersion)
    }
    
    return nil
}
```

</td>
</tr>
</table>

## Monitoring and Metrics

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class SyncMutationMetrics
{
    private readonly IMetrics _metrics;
    
    public void RecordMutation(EntityMutationRecord mutation)
    {
        _metrics.Increment("sync_mutation.recorded", new[]
        {
            ("entity_type", mutation.EntityType),
            ("mutation_type", mutation.MutationType.ToString()),
            ("sync_status", mutation.SyncStatus.ToString())
        });
    }
    
    public void RecordSyncResult(string entityType, bool success, TimeSpan duration)
    {
        _metrics.Increment("sync_mutation.processed", new[]
        {
            ("entity_type", entityType),
            ("success", success.ToString())
        });
        
        _metrics.Histogram("sync_mutation.duration", 
            duration.TotalMilliseconds, new[]
            {
                ("entity_type", entityType),
                ("success", success.ToString())
            });
    }
    
    public void RecordConflict(string entityType, string conflictType)
    {
        _metrics.Increment("sync_mutation.conflict", new[]
        {
            ("entity_type", entityType),
            ("conflict_type", conflictType)
        });
    }
}
```

</td>
<td>

```go
type SyncMutationMetrics struct {
    metrics metrics.Client
}

func (m *SyncMutationMetrics) RecordMutation(mutation *journal.EntityMutationRecord) {
    m.metrics.Increment("sync_mutation.recorded", metrics.Tags{
        "entity_type":   mutation.EntityType,
        "mutation_type": string(mutation.MutationType),
        "sync_status":   string(mutation.SyncStatus),
    })
}

func (m *SyncMutationMetrics) RecordSyncResult(
    entityType string, 
    success bool, 
    duration time.Duration) {
    
    successStr := fmt.Sprintf("%t", success)
    tags := metrics.Tags{
        "entity_type": entityType,
        "success":     successStr,
    }
    
    m.metrics.Increment("sync_mutation.processed", tags)
    m.metrics.Histogram("sync_mutation.duration", 
        float64(duration.Milliseconds()), tags)
}

func (m *SyncMutationMetrics) RecordConflict(entityType, conflictType string) {
    m.metrics.Increment("sync_mutation.conflict", metrics.Tags{
        "entity_type":   entityType,
        "conflict_type": conflictType,
    })
}

// Example usage with sync service
func (s *SyncService) syncMutationWithMetrics(
    ctx context.Context, 
    mutation *journal.EntityMutationRecord) error {
    
    start := time.Now()
    
    err := s.syncMutation(ctx, mutation)
    duration := time.Since(start)
    
    // Record metrics
    s.metrics.RecordSyncResult(mutation.EntityType, err == nil, duration)
    
    if err != nil {
        s.metrics.RecordConflict(mutation.EntityType, "sync_failed")
    }
    
    return err
}
```

</td>
</tr>
</table>

## Testing

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
[Test]
public async Task SyncMutationJournal_ShouldRecordAndQueryMutations()
{
    // Arrange
    var journal = CreateTestJournal();
    var order = new Order { Id = "123", Status = OrderStatus.Created };
    
    var mutation = new EntityMutationRecord
    {
        EntityType = "Order",
        EntityId = order.Id,
        EntityVersion = 1,
        MutationType = EntityMutationType.Create,
        CorrelationId = "corr-123",
        AfterState = JsonSerializer.Serialize(order),
        SyncStatus = SyncStatus.Pending
    };
    
    // Act
    await journal.RecordEntityMutationAsync(mutation);
    var mutations = await journal.GetEntityMutationsAsync("Order", order.Id);
    
    // Assert
    Assert.AreEqual(1, mutations.Count);
    var recorded = mutations.First();
    Assert.AreEqual("Order", recorded.EntityType);
    Assert.AreEqual(order.Id, recorded.EntityId);
    Assert.AreEqual(SyncStatus.Pending, recorded.SyncStatus);
}

[Test]
public async Task SyncMutationJournal_ShouldTrackSyncStatus()
{
    // Arrange
    var journal = CreateTestJournal();
    var mutation = await CreateTestMutation(journal);
    
    // Act - Mark as synced
    await journal.MarkAsSyncedAsync(
        new[] { mutation.Id }, 
        DateTime.UtcNow);
    
    var updatedMutations = await journal.GetEntityMutationsAsync(
        mutation.EntityType, mutation.EntityId);
    
    // Assert
    var updated = updatedMutations.First();
    Assert.AreEqual(SyncStatus.Synced, updated.SyncStatus);
    Assert.IsNotNull(updated.SyncedAt);
}
```

</td>
<td>

```go
func TestSyncMutationJournal_ShouldRecordAndQueryMutations(t *testing.T) {
    // Arrange
    journal := createTestJournal(t)
    order := &Order{ID: "123", Status: OrderStatusCreated}
    
    afterState, err := json.Marshal(order)
    require.NoError(t, err)
    
    mutation := &journal.EntityMutationRecord{
        EntityType:    "Order",
        EntityID:      order.ID,
        EntityVersion: 1,
        MutationType:  journal.EntityCreate,
        CorrelationID: "corr-123",
        AfterState:    json.RawMessage(afterState),
        SyncStatus:    journal.SyncStatusPending,
    }
    
    // Act
    err = journal.RecordEntityMutation(context.Background(), mutation)
    require.NoError(t, err)
    
    mutations, err := journal.GetEntityMutations(context.Background(), "Order", order.ID)
    require.NoError(t, err)
    
    // Assert
    assert.Len(t, mutations, 1)
    recorded := mutations[0]
    assert.Equal(t, "Order", recorded.EntityType)
    assert.Equal(t, order.ID, recorded.EntityID)
    assert.Equal(t, journal.SyncStatusPending, recorded.SyncStatus)
}

func TestSyncMutationJournal_ShouldTrackSyncStatus(t *testing.T) {
    // Arrange
    journal := createTestJournal(t)
    mutation := createTestMutation(t, journal)
    
    // Act - Mark as synced
    err := journal.MarkAsSynced(
        context.Background(),
        []string{mutation.ID}, 
        time.Now())
    require.NoError(t, err)
    
    updatedMutations, err := journal.GetEntityMutations(
        context.Background(),
        mutation.EntityType, 
        mutation.EntityID)
    require.NoError(t, err)
    
    // Assert
    assert.Len(t, updatedMutations, 1)
    updated := updatedMutations[0]
    assert.Equal(t, journal.SyncStatusSynced, updated.SyncStatus)
    assert.NotNil(t, updated.SyncedAt)
}

func createTestJournal(t *testing.T) journal.SyncMutationJournal {
    baseOpts := []journal.InMemoryJournalOption{
        journal.WithMaxEntries(1000),
    }
    syncOpts := []journal.SyncJournalOption{
        journal.WithServiceID("test-service"),
    }
    return journal.NewInMemorySyncMutationJournal(baseOpts, syncOpts...)
}
```

</td>
</tr>
</table>

## Best Practices

1. **Entity Modeling**
   - Use consistent entity type names across services
   - Include entity version for conflict detection
   - Store meaningful before/after states

2. **Correlation Management**
   - Use correlation IDs for transaction grouping
   - Include business context in correlation IDs
   - Maintain correlation ID consistency across service boundaries

3. **Sync Strategy**
   - Process mutations in entity version order
   - Implement idempotent sync operations
   - Use batch processing for efficiency

4. **Conflict Resolution**
   - Implement automatic conflict detection
   - Provide manual resolution workflows
   - Log all conflict resolutions for audit

5. **Performance**
   - Index by entity type and ID for fast queries
   - Regularly clean up old synced mutations
   - Monitor journal size and performance

## Troubleshooting

### Common Issues

**High Memory Usage**
- Monitor journal size and retention settings
- Implement regular cleanup of old mutations
- Consider database persistence for large volumes

**Sync Conflicts**
- Review entity versioning strategy
- Check clock synchronization between services
- Implement proper conflict resolution workflows

**Missing Mutations**
- Verify mutation recording in all write operations
- Check error handling in mutation recording
- Monitor for silent failures

### Debugging

```go
// Enable detailed sync mutation logging
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))

// Monitor journal statistics
journal := client.SyncMutationJournal()
if inMemoryJournal, ok := journal.(*journal.InMemorySyncMutationJournal); ok {
    stats, err := inMemoryJournal.GetStats(context.Background())
    if err == nil {
        log.Printf("Journal stats: %+v", stats)
    }
}
```