# Monitoring Component

The monitoring component provides comprehensive observability for Mmate messaging systems, including health checks, metrics, and monitoring tools.

> **Service-Scoped Monitoring**: Each microservice should monitor only its own resources to prevent "mastodon" patterns where one service monitors the entire cluster. See [Service-Scoped Monitoring](#service-scoped-monitoring) for implementation patterns.

## Overview

The monitoring component offers:
- Real-time queue and exchange monitoring
- Connection and channel health checks
- Message flow metrics
- Consumer lag tracking
- Dead letter queue monitoring
- Performance metrics
- CLI and TUI monitoring tools

## Health Monitoring

### Basic Health Checks

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure health checks
services.AddHealthChecks()
    .AddRabbitMQ(
        rabbitConnectionString: "amqp://localhost",
        name: "rabbitmq",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "messaging", "infrastructure" })
    .AddCheck<MessageProcessingHealthCheck>(
        "message-processing",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "messaging", "processing" });

// Custom health check
public class MessageProcessingHealthCheck : IHealthCheck
{
    private readonly IMessageMonitor _monitor;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken)
    {
        var stats = await _monitor.GetStatsAsync();
        
        // Check error rate
        var errorRate = stats.ErrorCount / 
            (double)stats.TotalProcessed;
        
        if (errorRate > 0.05) // 5% error threshold
        {
            return HealthCheckResult.Unhealthy(
                $"High error rate: {errorRate:P}");
        }
        
        // Check processing lag
        if (stats.QueueDepth > 1000)
        {
            return HealthCheckResult.Degraded(
                $"High queue depth: {stats.QueueDepth}");
        }
        
        return HealthCheckResult.Healthy(
            $"Processing {stats.MessagesPerSecond} msg/s");
    }
}
```

</td>
<td>

```go
import "github.com/glimte/mmate-go"

// Service-scoped health monitoring (RECOMMENDED)
func setupServiceHealth(client *mmate.Client, connectionString string) {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
        defer cancel()
        
        // Check health of THIS service's resources only
        health, err := client.GetServiceHealth(ctx)
        if err != nil {
            http.Error(w, err.Error(), http.StatusServiceUnavailable)
            return
        }
        
        statusCode := http.StatusOK
        if health.Status != "healthy" {
            statusCode = http.StatusServiceUnavailable
        }
        
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(statusCode)
        json.NewEncoder(w).Encode(health)
    })
}

// Custom health checks with service monitor
func advancedHealthCheck(client *mmate.Client) error {
    serviceMonitor, err := client.NewServiceMonitor()
    if err != nil {
        return err
    }
    
    ctx := context.Background()
    
    // Check this service's queue health only
    queueInfo, err := serviceMonitor.ServiceQueueInfo(ctx)
    if err != nil {
        return fmt.Errorf("queue health check failed: %w", err)
    }
    
    // Apply service-specific thresholds
    if queueInfo.Messages > 1000 {
        return fmt.Errorf("queue depth too high: %d messages", queueInfo.Messages)
    }
    
    return nil
}

func (h *HealthChecker) checkProcessing(
    ctx context.Context) CheckResult {
    
    stats, err := h.monitor.GetStats(ctx)
    if err != nil {
        return CheckResult{
            Status:  HealthStatusUnhealthy,
            Message: fmt.Sprintf("Failed to get stats: %v", err),
        }
    }
    
    // Check error rate
    errorRate := float64(stats.ErrorCount) / 
        float64(stats.TotalProcessed)
    
    if errorRate > 0.05 { // 5% threshold
        return CheckResult{
            Status:  HealthStatusUnhealthy,
            Message: fmt.Sprintf("High error rate: %.2f%%", 
                errorRate*100),
        }
    }
    
    // Check queue depth
    if stats.QueueDepth > 1000 {
        return CheckResult{
            Status:  HealthStatusDegraded,
            Message: fmt.Sprintf("High queue depth: %d", 
                stats.QueueDepth),
        }
    }
    
    return CheckResult{
        Status:  HealthStatusHealthy,
        Message: fmt.Sprintf("Processing %.2f msg/s", 
            stats.MessagesPerSecond),
    }
}
```

</td>
</tr>
</table>

### Health Check Endpoints

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Map health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Liveness - no checks
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("infrastructure")
});

// Custom response format
app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        
        var response = new
        {
            status = report.Status.ToString(),
            duration = report.TotalDuration.TotalMilliseconds,
            timestamp = DateTime.UtcNow,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds,
                description = e.Value.Description,
                exception = e.Value.Exception?.Message,
                data = e.Value.Data
            })
        };
        
        await context.Response.WriteAsync(
            JsonSerializer.Serialize(response));
    }
});
```

</td>
<td>

```go
// Health check HTTP handler
func (h *HealthHandler) ServeHTTP(
    w http.ResponseWriter, r *http.Request) {
    
    ctx := r.Context()
    
    switch r.URL.Path {
    case "/health":
        h.handleHealth(w, r)
    case "/health/live":
        h.handleLiveness(w, r)
    case "/health/ready":
        h.handleReadiness(w, r)
    case "/health/detail":
        h.handleDetail(w, r)
    default:
        http.NotFound(w, r)
    }
}

func (h *HealthHandler) handleDetail(
    w http.ResponseWriter, r *http.Request) {
    
    ctx := r.Context()
    start := time.Now()
    
    status, err := h.checker.CheckHealth(ctx)
    if err != nil {
        h.writeError(w, err)
        return
    }
    
    response := DetailedHealthResponse{
        Status:    status.Status,
        Duration:  time.Since(start).Milliseconds(),
        Timestamp: time.Now(),
        Checks:    make([]CheckDetail, 0),
    }
    
    for name, check := range status.Checks {
        response.Checks = append(response.Checks, 
            CheckDetail{
                Name:        name,
                Status:      check.Status,
                Duration:    check.Duration.Milliseconds(),
                Description: check.Message,
                Error:       check.Error,
                Data:        check.Data,
            })
    }
    
    // Set status code
    switch status.Status {
    case HealthStatusHealthy:
        w.WriteHeader(http.StatusOK)
    case HealthStatusDegraded:
        w.WriteHeader(http.StatusOK)
    case HealthStatusUnhealthy:
        w.WriteHeader(http.StatusServiceUnavailable)
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

</td>
</tr>
</table>

## Queue Monitoring

### Queue Metrics

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class QueueMonitor : IQueueMonitor
{
    private readonly IManagementClient _management;
    
    public async Task<QueueStats> GetQueueStatsAsync(
        string queueName)
    {
        var queue = await _management
            .GetQueueAsync(queueName);
        
        return new QueueStats
        {
            Name = queue.Name,
            Messages = queue.Messages,
            MessagesReady = queue.MessagesReady,
            MessagesUnacknowledged = queue.MessagesUnacknowledged,
            
            MessageStats = new MessageStats
            {
                PublishRate = queue.MessageStats?.PublishDetails?.Rate ?? 0,
                DeliverRate = queue.MessageStats?.DeliverGetDetails?.Rate ?? 0,
                AckRate = queue.MessageStats?.AckDetails?.Rate ?? 0,
                RedeliverRate = queue.MessageStats?.RedeliverDetails?.Rate ?? 0
            },
            
            Consumers = queue.Consumers,
            ConsumerUtilization = queue.ConsumerUtilisation,
            
            Memory = queue.Memory,
            
            IdleSince = queue.IdleSince,
            
            Policy = queue.EffectivePolicy,
            
            BackingQueueStatus = new BackingQueueStatus
            {
                Mode = queue.BackingQueueStatus?.Mode,
                Q1 = queue.BackingQueueStatus?.Q1 ?? 0,
                Q2 = queue.BackingQueueStatus?.Q2 ?? 0,
                Q3 = queue.BackingQueueStatus?.Q3 ?? 0,
                Q4 = queue.BackingQueueStatus?.Q4 ?? 0,
                Length = queue.BackingQueueStatus?.Len ?? 0,
                PersistentCount = queue.BackingQueueStatus?.PersistentCount ?? 0
            }
        };
    }
    
    public async Task<IEnumerable<QueueAlert>> 
        CheckQueuesAsync()
    {
        var alerts = new List<QueueAlert>();
        var queues = await _management.GetQueuesAsync();
        
        foreach (var queue in queues)
        {
            // High queue depth
            if (queue.Messages > 10000)
            {
                alerts.Add(new QueueAlert
                {
                    Queue = queue.Name,
                    Level = AlertLevel.Warning,
                    Message = $"High message count: {queue.Messages}"
                });
            }
            
            // No consumers
            if (queue.Consumers == 0 && queue.Messages > 0)
            {
                alerts.Add(new QueueAlert
                {
                    Queue = queue.Name,
                    Level = AlertLevel.Error,
                    Message = "No consumers for non-empty queue"
                });
            }
            
            // Low consumer utilization
            if (queue.ConsumerUtilisation < 0.5 && 
                queue.Consumers > 0)
            {
                alerts.Add(new QueueAlert
                {
                    Queue = queue.Name,
                    Level = AlertLevel.Warning,
                    Message = $"Low consumer utilization: {queue.ConsumerUtilisation:P}"
                });
            }
        }
        
        return alerts;
    }
}
```

</td>
<td>

```go
type QueueMonitor struct {
    client *monitor.Client
}

func (m *QueueMonitor) GetQueueStats(
    ctx context.Context, 
    queueName string) (*QueueStats, error) {
    
    queue, err := m.client.GetQueue(ctx, queueName)
    if err != nil {
        return nil, err
    }
    
    return &QueueStats{
        Name:                   queue.Name,
        Messages:              queue.Messages,
        MessagesReady:         queue.MessagesReady,
        MessagesUnacknowledged: queue.MessagesUnacknowledged,
        
        MessageStats: MessageStats{
            PublishRate:   queue.MessageStats.PublishDetails.Rate,
            DeliverRate:   queue.MessageStats.DeliverGetDetails.Rate,
            AckRate:       queue.MessageStats.AckDetails.Rate,
            RedeliverRate: queue.MessageStats.RedeliverDetails.Rate,
        },
        
        Consumers:           queue.Consumers,
        ConsumerUtilization: queue.ConsumerUtilisation,
        
        Memory: queue.Memory,
        
        IdleSince: queue.IdleSince,
        
        Policy: queue.EffectivePolicy,
        
        BackingQueueStatus: BackingQueueStatus{
            Mode:            queue.BackingQueueStatus.Mode,
            Q1:              queue.BackingQueueStatus.Q1,
            Q2:              queue.BackingQueueStatus.Q2,
            Q3:              queue.BackingQueueStatus.Q3,
            Q4:              queue.BackingQueueStatus.Q4,
            Length:          queue.BackingQueueStatus.Len,
            PersistentCount: queue.BackingQueueStatus.PersistentCount,
        },
    }, nil
}

func (m *QueueMonitor) CheckQueues(
    ctx context.Context) ([]QueueAlert, error) {
    
    queues, err := m.client.ListQueues(ctx)
    if err != nil {
        return nil, err
    }
    
    var alerts []QueueAlert
    
    for _, queue := range queues {
        // High queue depth
        if queue.Messages > 10000 {
            alerts = append(alerts, QueueAlert{
                Queue:   queue.Name,
                Level:   AlertLevelWarning,
                Message: fmt.Sprintf("High message count: %d", 
                    queue.Messages),
            })
        }
        
        // No consumers
        if queue.Consumers == 0 && queue.Messages > 0 {
            alerts = append(alerts, QueueAlert{
                Queue:   queue.Name,
                Level:   AlertLevelError,
                Message: "No consumers for non-empty queue",
            })
        }
        
        // Low consumer utilization
        if queue.ConsumerUtilisation < 0.5 && 
           queue.Consumers > 0 {
            alerts = append(alerts, QueueAlert{
                Queue:   queue.Name,
                Level:   AlertLevelWarning,
                Message: fmt.Sprintf(
                    "Low consumer utilization: %.2f%%",
                    queue.ConsumerUtilisation*100),
            })
        }
    }
    
    return alerts, nil
}
```

</td>
</tr>
</table>

### Consumer Monitoring

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class ConsumerMonitor : IConsumerMonitor
{
    public async Task<ConsumerStats> GetConsumerStatsAsync(
        string queueName)
    {
        var consumers = await _management
            .GetConsumersAsync(queueName);
        
        var stats = new ConsumerStats
        {
            QueueName = queueName,
            TotalConsumers = consumers.Count(),
            ActiveConsumers = consumers.Count(c => c.Active),
            Consumers = new List<ConsumerInfo>()
        };
        
        foreach (var consumer in consumers)
        {
            stats.Consumers.Add(new ConsumerInfo
            {
                Tag = consumer.ConsumerTag,
                ConnectionName = consumer.Channel.ConnectionName,
                ChannelNumber = consumer.Channel.Number,
                PrefetchCount = consumer.PrefetchCount,
                Exclusive = consumer.Exclusive,
                AckRequired = consumer.AckRequired,
                
                MessagesConsumed = consumer.MessageStats?.Deliver ?? 0,
                MessagesAcked = consumer.MessageStats?.Ack ?? 0,
                MessagesRedelivered = consumer.MessageStats?.Redeliver ?? 0,
                
                DeliverRate = consumer.MessageStats?.DeliverDetails?.Rate ?? 0,
                AckRate = consumer.MessageStats?.AckDetails?.Rate ?? 0
            });
        }
        
        // Calculate consumer lag
        var queueStats = await GetQueueStatsAsync(queueName);
        stats.ConsumerLag = queueStats.MessagesReady;
        
        if (stats.ActiveConsumers > 0)
        {
            stats.AverageProcessingRate = 
                stats.Consumers.Sum(c => c.DeliverRate);
            
            if (stats.AverageProcessingRate > 0)
            {
                stats.EstimatedTimeToEmpty = TimeSpan.FromSeconds(
                    stats.ConsumerLag / stats.AverageProcessingRate);
            }
        }
        
        return stats;
    }
}
```

</td>
<td>

```go
type ConsumerMonitor struct {
    client *monitor.Client
}

func (m *ConsumerMonitor) GetConsumerStats(
    ctx context.Context,
    queueName string) (*ConsumerStats, error) {
    
    consumers, err := m.client.GetConsumers(ctx, queueName)
    if err != nil {
        return nil, err
    }
    
    stats := &ConsumerStats{
        QueueName:      queueName,
        TotalConsumers: len(consumers),
        Consumers:      make([]ConsumerInfo, 0),
    }
    
    for _, consumer := range consumers {
        if consumer.Active {
            stats.ActiveConsumers++
        }
        
        info := ConsumerInfo{
            Tag:            consumer.ConsumerTag,
            ConnectionName: consumer.Channel.ConnectionName,
            ChannelNumber:  consumer.Channel.Number,
            PrefetchCount:  consumer.PrefetchCount,
            Exclusive:      consumer.Exclusive,
            AckRequired:    consumer.AckRequired,
            
            MessagesConsumed:    consumer.MessageStats.Deliver,
            MessagesAcked:      consumer.MessageStats.Ack,
            MessagesRedelivered: consumer.MessageStats.Redeliver,
            
            DeliverRate: consumer.MessageStats.DeliverDetails.Rate,
            AckRate:     consumer.MessageStats.AckDetails.Rate,
        }
        
        stats.Consumers = append(stats.Consumers, info)
    }
    
    // Calculate consumer lag
    queueStats, err := m.GetQueueStats(ctx, queueName)
    if err != nil {
        return nil, err
    }
    
    stats.ConsumerLag = queueStats.MessagesReady
    
    if stats.ActiveConsumers > 0 {
        for _, c := range stats.Consumers {
            stats.AverageProcessingRate += c.DeliverRate
        }
        
        if stats.AverageProcessingRate > 0 {
            seconds := float64(stats.ConsumerLag) / 
                stats.AverageProcessingRate
            stats.EstimatedTimeToEmpty = 
                time.Duration(seconds) * time.Second
        }
    }
    
    return stats, nil
}
```

</td>
</tr>
</table>

## Dead Letter Queue Monitoring

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class DLQMonitor : IDLQMonitor
{
    public async Task<DLQStats> GetDLQStatsAsync()
    {
        var dlqPattern = "dlq.*";
        var queues = await _management.GetQueuesAsync();
        var dlQueues = queues.Where(q => 
            q.Name.StartsWith("dlq."));
        
        var stats = new DLQStats
        {
            TotalDLQs = dlQueues.Count(),
            TotalMessages = dlQueues.Sum(q => q.Messages),
            DLQs = new List<DLQInfo>()
        };
        
        foreach (var dlq in dlQueues)
        {
            var info = new DLQInfo
            {
                Name = dlq.Name,
                OriginalQueue = dlq.Name.Substring(4), // Remove "dlq."
                MessageCount = dlq.Messages,
                OldestMessage = await GetOldestMessageAge(dlq.Name),
                IncomingRate = dlq.MessageStats?.PublishDetails?.Rate ?? 0
            };
            
            // Sample messages for analysis
            var messages = await SampleMessages(dlq.Name, 10);
            info.ErrorTypes = messages
                .GroupBy(m => m.Headers["x-death-reason"])
                .Select(g => new ErrorTypeCount
                {
                    Reason = g.Key,
                    Count = g.Count()
                })
                .ToList();
            
            stats.DLQs.Add(info);
        }
        
        return stats;
    }
    
    public async Task<RequeueResult> RequeueMessagesAsync(
        string dlqName, 
        int count = 0)
    {
        var result = new RequeueResult
        {
            DLQName = dlqName,
            RequestedCount = count
        };
        
        try
        {
            var messages = await GetMessages(dlqName, count);
            var originalQueue = dlqName.Substring(4);
            
            foreach (var message in messages)
            {
                try
                {
                    // Remove death headers
                    message.Headers.Remove("x-death");
                    message.Headers.Remove("x-death-reason");
                    
                    // Republish to original queue
                    await _publisher.PublishAsync(
                        message,
                        exchange: "",
                        routingKey: originalQueue);
                    
                    // Acknowledge from DLQ
                    await AcknowledgeMessage(dlqName, message.DeliveryTag);
                    
                    result.SuccessCount++;
                }
                catch (Exception ex)
                {
                    result.FailedCount++;
                    result.Errors.Add($"Message {message.MessageId}: {ex.Message}");
                }
            }
            
            result.Success = result.FailedCount == 0;
        }
        catch (Exception ex)
        {
            result.Success = false;
            result.Errors.Add(ex.Message);
        }
        
        return result;
    }
}
```

</td>
<td>

```go
type DLQMonitor struct {
    client    *monitor.Client
    publisher messaging.Publisher
}

func (m *DLQMonitor) GetDLQStats(
    ctx context.Context) (*DLQStats, error) {
    
    queues, err := m.client.ListQueues(ctx)
    if err != nil {
        return nil, err
    }
    
    stats := &DLQStats{
        DLQs: make([]DLQInfo, 0),
    }
    
    for _, queue := range queues {
        if !strings.HasPrefix(queue.Name, "dlq.") {
            continue
        }
        
        stats.TotalDLQs++
        stats.TotalMessages += queue.Messages
        
        info := DLQInfo{
            Name:          queue.Name,
            OriginalQueue: strings.TrimPrefix(queue.Name, "dlq."),
            MessageCount:  queue.Messages,
            IncomingRate:  queue.MessageStats.PublishDetails.Rate,
        }
        
        // Get oldest message age
        if oldestAge, err := m.getOldestMessageAge(
            ctx, queue.Name); err == nil {
            info.OldestMessage = oldestAge
        }
        
        // Sample messages for error analysis
        messages, err := m.sampleMessages(ctx, queue.Name, 10)
        if err == nil {
            errorCounts := make(map[string]int)
            for _, msg := range messages {
                reason := msg.Headers["x-death-reason"]
                errorCounts[reason]++
            }
            
            for reason, count := range errorCounts {
                info.ErrorTypes = append(info.ErrorTypes,
                    ErrorTypeCount{
                        Reason: reason,
                        Count:  count,
                    })
            }
        }
        
        stats.DLQs = append(stats.DLQs, info)
    }
    
    return stats, nil
}

func (m *DLQMonitor) RequeueMessages(
    ctx context.Context,
    dlqName string,
    count int) (*RequeueResult, error) {
    
    result := &RequeueResult{
        DLQName:        dlqName,
        RequestedCount: count,
        Errors:         make([]string, 0),
    }
    
    messages, err := m.getMessages(ctx, dlqName, count)
    if err != nil {
        result.Success = false
        result.Errors = append(result.Errors, err.Error())
        return result, err
    }
    
    originalQueue := strings.TrimPrefix(dlqName, "dlq.")
    
    for _, msg := range messages {
        // Remove death headers
        delete(msg.Headers, "x-death")
        delete(msg.Headers, "x-death-reason")
        
        // Republish to original queue
        err := m.publisher.Publish(ctx, msg,
            messaging.WithExchange(""),
            messaging.WithRoutingKey(originalQueue))
        
        if err != nil {
            result.FailedCount++
            result.Errors = append(result.Errors,
                fmt.Sprintf("Message %s: %v", 
                    msg.MessageID, err))
            continue
        }
        
        // Acknowledge from DLQ
        err = m.acknowledgeMessage(ctx, dlqName, 
            msg.DeliveryTag)
        if err != nil {
            result.FailedCount++
            result.Errors = append(result.Errors,
                fmt.Sprintf("Ack failed for %s: %v",
                    msg.MessageID, err))
            continue
        }
        
        result.SuccessCount++
    }
    
    result.Success = result.FailedCount == 0
    return result, nil
}
```

</td>
</tr>
</table>

## Performance Metrics

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
public class PerformanceMonitor : IPerformanceMonitor
{
    private readonly IMetricsCollector _metrics;
    
    public async Task<PerformanceStats> GetStatsAsync(
        TimeSpan period)
    {
        var endTime = DateTime.UtcNow;
        var startTime = endTime - period;
        
        var stats = new PerformanceStats
        {
            Period = period,
            StartTime = startTime,
            EndTime = endTime
        };
        
        // Message throughput
        stats.MessagesPublished = await _metrics
            .GetCounterValueAsync("messages.published", period);
        stats.MessagesConsumed = await _metrics
            .GetCounterValueAsync("messages.consumed", period);
        stats.MessagesErrored = await _metrics
            .GetCounterValueAsync("messages.errors", period);
        
        stats.PublishRate = stats.MessagesPublished / 
            period.TotalSeconds;
        stats.ConsumeRate = stats.MessagesConsumed / 
            period.TotalSeconds;
        stats.ErrorRate = stats.MessagesErrored / 
            period.TotalSeconds;
        
        // Latency percentiles
        var latencies = await _metrics
            .GetHistogramPercentilesAsync(
                "message.processing.duration", period);
        
        stats.ProcessingLatency = new LatencyStats
        {
            P50 = latencies[50],
            P75 = latencies[75],
            P90 = latencies[90],
            P95 = latencies[95],
            P99 = latencies[99],
            P999 = latencies[99.9],
            Max = latencies[100]
        };
        
        // Resource usage
        var process = Process.GetCurrentProcess();
        stats.ResourceUsage = new ResourceStats
        {
            CpuUsage = await GetCpuUsageAsync(),
            MemoryUsed = process.WorkingSet64,
            ThreadCount = process.Threads.Count,
            HandleCount = process.HandleCount,
            
            ConnectionCount = await _metrics
                .GetGaugeValueAsync("rabbitmq.connections"),
            ChannelCount = await _metrics
                .GetGaugeValueAsync("rabbitmq.channels")
        };
        
        return stats;
    }
}
```

</td>
<td>

```go
type PerformanceMonitor struct {
    metrics MetricsCollector
}

func (m *PerformanceMonitor) GetStats(
    ctx context.Context,
    period time.Duration) (*PerformanceStats, error) {
    
    endTime := time.Now()
    startTime := endTime.Add(-period)
    
    stats := &PerformanceStats{
        Period:    period,
        StartTime: startTime,
        EndTime:   endTime,
    }
    
    // Message throughput
    published, err := m.metrics.GetCounterValue(
        "messages.published", period)
    if err != nil {
        return nil, err
    }
    stats.MessagesPublished = published
    
    consumed, err := m.metrics.GetCounterValue(
        "messages.consumed", period)
    if err != nil {
        return nil, err
    }
    stats.MessagesConsumed = consumed
    
    errored, err := m.metrics.GetCounterValue(
        "messages.errors", period)
    if err != nil {
        return nil, err
    }
    stats.MessagesErrored = errored
    
    // Calculate rates
    seconds := period.Seconds()
    stats.PublishRate = float64(published) / seconds
    stats.ConsumeRate = float64(consumed) / seconds
    stats.ErrorRate = float64(errored) / seconds
    
    // Latency percentiles
    latencies, err := m.metrics.GetHistogramPercentiles(
        "message.processing.duration", period)
    if err != nil {
        return nil, err
    }
    
    stats.ProcessingLatency = LatencyStats{
        P50:  latencies[50.0],
        P75:  latencies[75.0],
        P90:  latencies[90.0],
        P95:  latencies[95.0],
        P99:  latencies[99.0],
        P999: latencies[99.9],
        Max:  latencies[100.0],
    }
    
    // Resource usage
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    
    stats.ResourceUsage = ResourceStats{
        CpuUsage:    getCPUUsage(),
        MemoryUsed:  memStats.Alloc,
        GoroutineCount: runtime.NumGoroutine(),
        
        ConnectionCount: m.metrics.GetGaugeValue(
            "rabbitmq.connections"),
        ChannelCount: m.metrics.GetGaugeValue(
            "rabbitmq.channels"),
    }
    
    return stats, nil
}
```

</td>
</tr>
</table>

## Monitoring Tools

### CLI Monitor

The CLI monitor provides command-line access to monitoring data:

```bash
# Show overview
mmate monitor overview

# List queues
mmate monitor queues --sort=depth --limit=10

# Watch specific queue
mmate monitor queue orders --watch --interval=5s

# Show consumers
mmate monitor consumers --queue=orders

# Check DLQ
mmate monitor dlq --stats
mmate monitor dlq --requeue=dlq.orders --count=100

# Performance stats
mmate monitor perf --period=1h

# Export metrics
mmate monitor export --format=prometheus
mmate monitor export --format=json --output=metrics.json
```

### TUI Dashboard

The TUI (Terminal UI) provides an interactive dashboard:

```bash
# Start TUI
mmate-tui

# Connect to specific instance
mmate-tui --url=amqp://user:pass@host:5672

# Custom refresh interval
mmate-tui --refresh=2s
```

TUI Features:
- Real-time queue statistics
- Consumer monitoring
- Message flow visualization
- Alert notifications
- Performance graphs
- Keyboard navigation

## Alert Configuration

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure alerts
services.Configure<AlertOptions>(options =>
{
    options.Rules.Add(new AlertRule
    {
        Name = "HighQueueDepth",
        Condition = ctx => ctx.Queue.Messages > 10000,
        Level = AlertLevel.Warning,
        Message = "Queue {QueueName} has {Messages} messages"
    });
    
    options.Rules.Add(new AlertRule
    {
        Name = "NoConsumers",
        Condition = ctx => ctx.Queue.Consumers == 0 && 
                          ctx.Queue.Messages > 0,
        Level = AlertLevel.Error,
        Message = "Queue {QueueName} has no consumers"
    });
    
    options.Rules.Add(new AlertRule
    {
        Name = "HighErrorRate",
        Condition = ctx => ctx.ErrorRate > 0.05,
        Level = AlertLevel.Critical,
        Message = "Error rate {ErrorRate:P} exceeds threshold"
    });
});

// Alert handler
services.AddSingleton<IAlertHandler, EmailAlertHandler>();
services.AddSingleton<IAlertHandler, SlackAlertHandler>();
services.AddSingleton<IAlertHandler, PagerDutyAlertHandler>();
```

</td>
<td>

```go
// Configure alerts
type AlertConfig struct {
    Rules []AlertRule
}

type AlertRule struct {
    Name      string
    Condition func(ctx AlertContext) bool
    Level     AlertLevel
    Message   string
}

// Default rules
var defaultAlertRules = []AlertRule{
    {
        Name: "HighQueueDepth",
        Condition: func(ctx AlertContext) bool {
            return ctx.Queue.Messages > 10000
        },
        Level:   AlertLevelWarning,
        Message: "Queue %s has %d messages",
    },
    {
        Name: "NoConsumers",
        Condition: func(ctx AlertContext) bool {
            return ctx.Queue.Consumers == 0 && 
                   ctx.Queue.Messages > 0
        },
        Level:   AlertLevelError,
        Message: "Queue %s has no consumers",
    },
    {
        Name: "HighErrorRate",
        Condition: func(ctx AlertContext) bool {
            return ctx.ErrorRate > 0.05
        },
        Level:   AlertLevelCritical,
        Message: "Error rate %.2f%% exceeds threshold",
    },
}

// Alert handlers
type AlertManager struct {
    handlers []AlertHandler
}

func NewAlertManager() *AlertManager {
    return &AlertManager{
        handlers: []AlertHandler{
            NewEmailAlertHandler(),
            NewSlackAlertHandler(),
            NewPagerDutyAlertHandler(),
        },
    }
}
```

</td>
</tr>
</table>

## Integration with Monitoring Systems

### Prometheus

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```csharp
// Configure Prometheus
services.AddSingleton<IMetricServer>(sp =>
{
    var server = new MetricServer(port: 9090);
    server.Start();
    return server;
});

// Custom metrics
public class PrometheusMetrics
{
    private readonly Counter _messagesProcessed;
    private readonly Histogram _processingDuration;
    private readonly Gauge _queueDepth;
    
    public PrometheusMetrics()
    {
        _messagesProcessed = Metrics
            .CreateCounter("mmate_messages_processed_total",
                "Total messages processed",
                new CounterConfiguration
                {
                    LabelNames = new[] { "queue", "type", "status" }
                });
        
        _processingDuration = Metrics
            .CreateHistogram("mmate_processing_duration_seconds",
                "Message processing duration",
                new HistogramConfiguration
                {
                    LabelNames = new[] { "queue", "type" },
                    Buckets = Histogram.ExponentialBuckets(0.001, 2, 10)
                });
        
        _queueDepth = Metrics
            .CreateGauge("mmate_queue_depth",
                "Current queue depth",
                new GaugeConfiguration
                {
                    LabelNames = new[] { "queue" }
                });
    }
}
```

</td>
<td>

```go
// Prometheus metrics
var (
    messagesProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "mmate_messages_processed_total",
            Help: "Total messages processed",
        },
        []string{"queue", "type", "status"},
    )
    
    processingDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "mmate_processing_duration_seconds",
            Help:    "Message processing duration",
            Buckets: prometheus.ExponentialBuckets(0.001, 2, 10),
        },
        []string{"queue", "type"},
    )
    
    queueDepth = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "mmate_queue_depth",
            Help: "Current queue depth",
        },
        []string{"queue"},
    )
)

func init() {
    prometheus.MustRegister(messagesProcessed)
    prometheus.MustRegister(processingDuration)
    prometheus.MustRegister(queueDepth)
}

// Expose metrics
http.Handle("/metrics", promhttp.Handler())
```

</td>
</tr>
</table>

### Grafana Dashboard

Example Grafana queries:

```promql
# Message rate
rate(mmate_messages_processed_total[5m])

# Error rate
rate(mmate_messages_processed_total{status="error"}[5m]) / 
rate(mmate_messages_processed_total[5m])

# Processing latency
histogram_quantile(0.95,
  rate(mmate_processing_duration_seconds_bucket[5m]))

# Queue depth over time
mmate_queue_depth

# Consumer lag
mmate_queue_depth{queue=~".*"} / 
rate(mmate_messages_processed_total{queue=~".*"}[5m])
```

## Service-Scoped Monitoring

### Philosophy: Avoid the "Mastodon" Anti-Pattern

Each microservice should monitor **only its own resources** to maintain proper service boundaries and prevent one service from becoming a "mastodon" that monitors the entire cluster.

#### ✅ What Each Service Should Monitor
- Its own message processing metrics (via interceptors)
- Its own queue(s) health
- Its own error rates and processing times
- Shared broker health (safe for all services)

#### ❌ What Services Should NOT Monitor
- Other services' queues or metrics
- Cluster-wide statistics
- Resources they don't own

### Go Implementation

<table>
<tr>
<th>Service-Scoped Approach</th>
<th>Mastodon Anti-Pattern</th>
</tr>
<tr>
<td>

```go
// ✅ GOOD: Service-scoped monitoring
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithServiceName("user-service"),
    mmate.WithDefaultMetrics(), // Only THIS service's metrics
)

// Monitor only this service's resources
serviceMetrics, err := client.GetServiceMetrics(ctx, connectionString)
serviceHealth, err := client.GetServiceHealth(ctx, connectionString)

// Set up service-scoped endpoints
http.HandleFunc("/metrics", func(w http.ResponseWriter, r *http.Request) {
    // Only exposes user-service metrics
    metrics, _ := client.GetServiceMetrics(r.Context(), connectionString)
    json.NewEncoder(w).Encode(metrics)
})
```

</td>
<td>

```go
// ❌ BAD: Direct HTTP management API access (removed from library)
// This anti-pattern has been removed to enforce service-scoped monitoring

// ✅ GOOD: Use service-scoped monitoring instead
health, err := client.GetServiceHealth(ctx)
metrics, err := client.GetServiceMetrics(ctx)
monitor, err := client.NewServiceMonitor()
```

</td>
</tr>
</table>

### Service Monitor for Advanced Use Cases

For advanced monitoring scenarios, use the `ServiceMonitor` to ensure service-scoped access:

```go
// Create service-scoped monitor
serviceMonitor, err := client.NewServiceMonitor(connectionString)

// Get this service's queue info only
queueInfo, err := serviceMonitor.ServiceQueueInfo(ctx)

// Get all queues owned by this service (naming convention)
serviceQueues, err := serviceMonitor.ServiceOwnedQueues(ctx)

// Safe: Check overall broker health (all services can do this)
brokerHealth, err := serviceMonitor.BrokerHealth(ctx)
```

### Multi-Service Architecture Pattern

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ User Service│    │Order Service│    │Pay Service  │
│   :8081     │    │   :8082     │    │   :8083     │
│ /metrics ✅ │    │ /metrics ✅ │    │ /metrics ✅ │
│ /health  ✅ │    │ /health  ✅ │    │ /health  ✅ │
└─────────────┘    └─────────────┘    └─────────────┘
      │                    │                    │
      │ Only own metrics   │ Only own metrics   │ Only own metrics
      └────────────────────┼────────────────────┘
                          │
               ┌─────────────────┐
               │ Monitoring Tool │
               │  (Prometheus)   │  ← Scrapes all service endpoints
               └─────────────────┘
```

### Service-Scoped Metrics Export

```go
// Each service exposes its own metrics in standard format
func (s *UserService) metricsHandler(w http.ResponseWriter, r *http.Request) {
    summary := s.client.GetMetricsSummary()
    
    // Export only this service's metrics
    for msgType, count := range summary.MessageCounts {
        fmt.Fprintf(w, "mmate_messages_total{service=\"user-service\",type=\"%s\"} %d\n", 
            msgType, count)
    }
    
    for msgType, stats := range summary.ProcessingStats {
        fmt.Fprintf(w, "mmate_processing_avg_ms{service=\"user-service\",type=\"%s\"} %d\n", 
            msgType, stats.AvgMs)
    }
}
```

### Naming Conventions for Service Isolation

To enable automatic service-scoped filtering:

```go
// Service queues follow naming convention:
// Main queue: "{service-name}-queue" 
// Additional: "{service-name}-{purpose}"

// User service owns:
// - user-service-queue (main)
// - user-service-dlq (dead letter)
// - user-service-retry (retry queue)

// ServiceMonitor automatically filters these
serviceQueues, err := serviceMonitor.ServiceOwnedQueues(ctx)
// Returns only queues starting with "user-service-"
```

## Best Practices

1. **Service Isolation**
   - Each service monitors only its own resources
   - Use ServiceMonitor for queue-level monitoring
   - Keep metrics collectors separate per service
   - Follow naming conventions for automatic filtering

2. **Monitoring Strategy**
   - Monitor both infrastructure and application metrics
   - Set up alerts for critical conditions
   - Track trends over time
   - Use dashboards for visualization

3. **Health Checks**
   - Implement multiple health check endpoints
   - Include dependency checks
   - Use appropriate failure thresholds
   - Consider graceful degradation

4. **Performance Monitoring**
   - Track message throughput
   - Monitor processing latency
   - Watch resource utilization
   - Identify bottlenecks

5. **Alert Management**
   - Define clear alert thresholds
   - Use alert levels appropriately
   - Avoid alert fatigue
   - Include actionable information

6. **DLQ Management**
   - Monitor DLQ growth
   - Analyze failure patterns
   - Implement requeue strategies
   - Set up aging alerts

## Troubleshooting

### Common Issues

1. **High Queue Depth**
   - Check consumer count
   - Verify consumer performance
   - Look for processing errors
   - Scale consumers if needed

2. **High Error Rate**
   - Check error logs
   - Analyze DLQ messages
   - Verify message format
   - Check downstream services

3. **Low Consumer Utilization**
   - Adjust prefetch count
   - Check network latency
   - Verify processing logic
   - Consider consumer placement

## Next Steps

- Set up [Prometheus Integration](#prometheus)
- Create [Grafana Dashboards](#grafana-dashboard)
- Configure [Alerts](#alert-configuration)
- Review [Performance Tuning](../../advanced/performance.md)