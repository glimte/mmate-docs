# Go API Reference

This document provides a complete API reference for the Mmate Go framework.

## Table of Contents

- [Contracts](#contracts)
- [Messaging](#messaging)
- [StageFlow](#stageflow)
- [Bridge](#bridge)
- [Interceptors](#interceptors)
- [Enterprise Features](#enterprise-features)
- [Monitoring](#monitoring)
- [Health](#health)

## Contracts

### Message Interface

```go
type Message interface {
    GetID() string
    GetType() string
    GetTimestamp() time.Time
    GetCorrelationID() string
    GetHeaders() map[string]string
    GetHeader(key string) string
    SetHeader(key, value string)
}
```

### Command

```go
type Command interface {
    Message
    Validate() error
}

// Base implementation
type BaseCommand struct {
    ID            string            `json:"id"`
    Type          string            `json:"type"`
    Timestamp     time.Time         `json:"timestamp"`
    CorrelationID string            `json:"correlationId,omitempty"`
    Headers       map[string]string `json:"headers,omitempty"`
}

// Constructor
func NewBaseCommand(messageType string) BaseCommand
```

### Event

```go
type Event interface {
    Message
    GetAggregateID() string
    GetVersion() int
}

// Base implementation
type BaseEvent struct {
    ID            string            `json:"id"`
    Type          string            `json:"type"`
    Timestamp     time.Time         `json:"timestamp"`
    CorrelationID string            `json:"correlationId,omitempty"`
    Headers       map[string]string `json:"headers,omitempty"`
    AggregateID   string            `json:"aggregateId"`
    Version       int               `json:"version"`
}

// Constructor
func NewBaseEvent(messageType, aggregateID string) BaseEvent
```

### Query

```go
type Query interface {
    Message
    GetReplyTo() string
}

// Base implementation
type BaseQuery struct {
    ID            string            `json:"id"`
    Type          string            `json:"type"`
    Timestamp     time.Time         `json:"timestamp"`
    CorrelationID string            `json:"correlationId,omitempty"`
    Headers       map[string]string `json:"headers,omitempty"`
    ReplyTo       string            `json:"replyTo"`
}

// Constructor
func NewBaseQuery(messageType string) BaseQuery
```

### Reply

```go
type Reply interface {
    Message
    IsSuccess() bool
    GetError() string
}

// Base implementation
type BaseReply struct {
    ID            string            `json:"id"`
    Type          string            `json:"type"`
    Timestamp     time.Time         `json:"timestamp"`
    CorrelationID string            `json:"correlationId,omitempty"`
    Headers       map[string]string `json:"headers,omitempty"`
    Success       bool              `json:"success"`
    Error         string            `json:"error,omitempty"`
}

// Constructor
func NewBaseReply(requestID, correlationID string) BaseReply
```

## Client

### Client Creation

```go
// Create client with default RabbitMQ transport
func NewClient(connectionString string) (*Client, error)

// Create client with options
func NewClientWithOptions(connectionString string, options ...ClientOption) (*Client, error)

// Client options
type ClientOption func(*clientConfig)

// Logging options
func WithLogger(logger *slog.Logger) ClientOption
func WithDefaultLogger() ClientOption

// Service configuration
func WithServiceName(name string) ClientOption
func WithFIFOMode(enabled bool) ClientOption

// Queue configuration
func WithQueueBindings(bindings ...messaging.QueueBinding) ClientOption

// Interceptor configuration
func WithInterceptors(pipeline *interceptors.Pipeline) ClientOption
func WithPublishInterceptors(pipeline *interceptors.Pipeline) ClientOption
func WithSubscribeInterceptors(pipeline *interceptors.Pipeline) ClientOption

// Reliability configuration
func WithRetryPolicy(policy reliability.RetryPolicy) ClientOption
func WithDefaultRetry() ClientOption
func WithCircuitBreaker(cb *reliability.CircuitBreaker) ClientOption
func WithDLQHandler(handler *reliability.DLQHandler) ClientOption

// Metrics configuration
func WithMetrics(collector interceptors.MetricsCollector) ClientOption
func WithDefaultMetrics() ClientOption

// Contract publishing configuration
func WithContractPublishing() ClientOption
```

### Client Methods

```go
type Client struct {
    // ... internal fields
}

// Core components
func (c *Client) Publisher() *MessagePublisher
func (c *Client) Subscriber() *MessageSubscriber
func (c *Client) Dispatcher() *MessageDispatcher
func (c *Client) Bridge() *bridge.SyncAsyncBridge
func (c *Client) Transport() Transport

// Service information
func (c *Client) ServiceQueue() string

// Metrics and monitoring
func (c *Client) MetricsCollector() interceptors.MetricsCollector
func (c *Client) GetMetricsSummary() *monitor.MetricsSummary
func (c *Client) NewServiceMonitor() (*monitor.ServiceMonitor, error)
func (c *Client) GetServiceMetrics(ctx context.Context) (*monitor.ServiceMetrics, error)
func (c *Client) GetServiceHealth(ctx context.Context) (*monitor.ServiceHealth, error)
func (c *Client) GetMyConsumerStats(ctx context.Context) (*monitor.ConsumerStats, error)
func (c *Client) GetAdvancedMetrics() *monitor.AdvancedMetricsReport

// Resource management
func (c *Client) Close() error
```

## Messaging

### MessagePublisher

```go
type MessagePublisher struct {
    // ... internal fields
}

// Constructor (typically accessed via Client.Publisher())
func NewMessagePublisher(transport TransportPublisher, opts ...PublisherOption) *MessagePublisher

// Publishing method
func (p *MessagePublisher) Publish(ctx context.Context, msg contracts.Message, options ...PublishOption) error

// Publisher options
type PublisherOption func(*MessagePublisher)

func WithPublisherLogger(logger *slog.Logger) PublisherOption
func WithCircuitBreaker(cb *CircuitBreaker) PublisherOption
func WithRetryPolicy(policy RetryPolicy) PublisherOption
func WithDefaultTTL(ttl time.Duration) PublisherOption

// Publish options
type PublishOption func(*PublishOptions)

func WithExchange(exchange string) PublishOption
func WithRoutingKey(routingKey string) PublishOption
func WithTTL(ttl time.Duration) PublishOption
func WithPriority(priority uint8) PublishOption
func WithPersistent(persistent bool) PublishOption
func WithHeaders(headers map[string]interface{}) PublishOption
func WithConfirmDelivery(confirm bool) PublishOption
func WithCorrelationID(correlationID string) PublishOption
func WithReplyTo(replyTo string) PublishOption
```

### MessageSubscriber

```go
type MessageSubscriber struct {
    // ... internal fields
}

// Constructor (typically accessed via Client.Subscriber())
func NewMessageSubscriber(transport TransportSubscriber, dispatcher *MessageDispatcher, opts ...SubscriberOption) *MessageSubscriber

// Subscribe to a queue
func (s *MessageSubscriber) Subscribe(ctx context.Context, queue string, messageType string, 
    handler MessageHandler, options ...SubscriptionOption) error

// Unsubscribe from a queue
func (s *MessageSubscriber) Unsubscribe(queue string) error

// Get active subscriptions
func (s *MessageSubscriber) GetSubscriptions() map[string]*Subscription

// Close all subscriptions
func (s *MessageSubscriber) Close() error

// Subscriber options
type SubscriberOption func(*MessageSubscriber)

func WithSubscriberLogger(logger *slog.Logger) SubscriberOption
func WithErrorHandler(errorHandler ErrorHandler) SubscriberOption
func WithDeadLetterQueue(dlq string) SubscriberOption

// Subscription options
type SubscriptionOption func(*SubscriptionOptions)

func WithPrefetchCount(count int) SubscriptionOption
func WithAutoAck(autoAck bool) SubscriptionOption
func WithSubscriberExclusive(exclusive bool) SubscriptionOption
func WithSubscriberDurable(durable bool) SubscriptionOption
func WithAutoDelete(autoDelete bool) SubscriptionOption
func WithMaxRetries(maxRetries int) SubscriptionOption
func WithDeadLetterExchange(exchange string) SubscriptionOption
```

### MessageHandler Interface

```go
type MessageHandler interface {
    Handle(ctx context.Context, msg Message) error
}

// Function adapter for handlers
type MessageHandlerFunc func(ctx context.Context, msg Message) error

func (f MessageHandlerFunc) Handle(ctx context.Context, msg Message) error {
    return f(ctx, msg)
}
```

### MessageDispatcher

```go
type MessageDispatcher struct {
    // ... internal fields
}

// Constructor (typically accessed via Client.Dispatcher())
func NewMessageDispatcher(options ...DispatcherOption) *MessageDispatcher

// Register a handler for a message type
func (d *MessageDispatcher) RegisterHandler(messageType string, handler MessageHandler, options ...HandlerOption) error

// Unregister a handler
func (d *MessageDispatcher) UnregisterHandler(messageType string) error

// Handle implements MessageHandler interface
func (d *MessageDispatcher) Handle(ctx context.Context, msg Message) error
```

### Transport Interfaces

```go
// Transport provides both publisher and subscriber functionality
type Transport interface {
    Publisher() TransportPublisher
    Subscriber() TransportSubscriber
    CreateQueue(ctx context.Context, name string, options QueueOptions) error
    DeleteQueue(ctx context.Context, name string) error
    BindQueue(ctx context.Context, queue, exchange, routingKey string) error
    Connect(ctx context.Context) error
    Close() error
    IsConnected() bool
}

// TransportPublisher defines the interface for publishing messages
type TransportPublisher interface {
    Publish(ctx context.Context, exchange, routingKey string, envelope *Envelope) error
    Close() error
}

// TransportSubscriber defines the interface for subscribing to messages
type TransportSubscriber interface {
    Subscribe(ctx context.Context, queue string, handler func(delivery TransportDelivery) error, 
        options SubscriptionOptions) error
    Unsubscribe(queue string) error
    Close() error
}

// TransportDelivery represents a message delivery
type TransportDelivery interface {
    Body() []byte
    Acknowledge() error
    Reject(requeue bool) error
    Headers() map[string]interface{}
}
```

### Error Types

```go
// Permanent error - message will be moved to DLQ
type PermanentError struct {
    Message string
    Cause   error
}

func NewPermanentError(message string, cause error) *PermanentError

// Transient error - message will be retried
type TransientError struct {
    Message    string
    Cause      error
    RetryAfter time.Duration
}

func NewTransientError(message string, cause error, opts ...TransientErrorOption) *TransientError
```

## StageFlow

### Flow

```go
type Flow[T any] struct {
    id      string
    stages  []Stage[T]
    options FlowOptions
}

// Constructor
func NewFlow[T any](id string, opts ...FlowOption) *Flow[T]

// Methods
func (f *Flow[T]) AddStage(name string, stage Stage[T]) *Flow[T]
func (f *Flow[T]) AddCompensation(stageName string, compensation CompensationStage[T]) *Flow[T]
func (f *Flow[T]) Execute(ctx context.Context, context T) (*FlowResult, error)
func (f *Flow[T]) ExecuteAsync(ctx context.Context, context T) (<-chan *FlowResult, error)

// Options
type FlowOption func(*FlowOptions)

func WithTimeout(timeout time.Duration) FlowOption
func WithMaxRetries(retries int) FlowOption
func WithCheckpoint(enabled bool) FlowOption
```

### Stage Interface

```go
type Stage[T any] interface {
    Execute(ctx context.Context, context T) error
}

type CompensationStage[T any] interface {
    Compensate(ctx context.Context, context T, stageError error) error
}

// TypedCompensationHandler for typed workflows
type TypedCompensationHandler[T any] interface {
    Compensate(ctx context.Context, context T, stageError error) error
    GetStageID() string
}
```

### FlowResult

```go
type FlowResult struct {
    Success       bool          `json:"success"`
    CompletedAt   time.Time     `json:"completedAt"`
    Duration      time.Duration `json:"duration"`
    Error         error         `json:"error,omitempty"`
    FailedStage   string        `json:"failedStage,omitempty"`
    Compensations []string      `json:"compensations,omitempty"`
}
```

### Compensation Workflow Types

```go
// CompensationMessageEnvelope wraps compensation messages
type CompensationMessageEnvelope struct {
    WorkflowID       string                 `json:"workflowId"`
    WorkflowInstance string                 `json:"workflowInstance"`
    FailedStage      string                 `json:"failedStage"`
    Error           string                 `json:"error"`
    Context         map[string]interface{} `json:"context"`
    CompletedStages []StageCompletionInfo  `json:"completedStages"`
    Timestamp       time.Time              `json:"timestamp"`
}

// StageCompletionInfo tracks completed stages for compensation
type StageCompletionInfo struct {
    StageID     string    `json:"stageId"`
    CompletedAt time.Time `json:"completedAt"`
    Result      string    `json:"result,omitempty"`
}
```

## Enterprise Features

### TTL Retry Scheduler

Persistent retry scheduling using RabbitMQ Dead Letter Exchange (DLX) mechanism.

```go
// TTLRetryScheduler provides persistent retry scheduling
type TTLRetryScheduler struct {
    channelPool      *rabbitmq.ChannelPool
    topologyManager  *rabbitmq.TopologyManager
    logger          *slog.Logger
    retryExchange   string
    delayExchange   string
}

// TTLRetrySchedulerOptions configures the TTL retry scheduler
type TTLRetrySchedulerOptions struct {
    RetryExchange string
    DelayExchange string
    Logger        *slog.Logger
}

// Create TTL retry scheduler
func NewTTLRetryScheduler(pool *rabbitmq.ChannelPool, opts *TTLRetrySchedulerOptions) *TTLRetryScheduler

// Initialize TTL retry topology
func (s *TTLRetryScheduler) Initialize(ctx context.Context) error

// Schedule a retry with persistent TTL
func (s *TTLRetryScheduler) ScheduleRetry(
    ctx context.Context, 
    msg contracts.Message, 
    originalQueue string, 
    attempt int, 
    policy RetryPolicy, 
    delay time.Duration, 
    lastErr error) error

// TTL Retry Interceptor
type TTLRetryInterceptor struct {
    scheduler   *TTLRetryScheduler
    retryPolicy RetryPolicy
    queueName   string
    logger      *slog.Logger
}

func NewTTLRetryInterceptor(scheduler *TTLRetryScheduler, opts *TTLRetryInterceptorOptions) *TTLRetryInterceptor
```

### Acknowledgment Tracking

Application-level acknowledgment tracking with correlation ID management.

```go
// ProcessingAcknowledgment represents an application-level processing acknowledgment
type ProcessingAcknowledgment struct {
    CorrelationID   string                 `json:"correlationId"`
    MessageID       string                 `json:"messageId"`
    MessageType     string                 `json:"messageType"`
    Success         bool                   `json:"success"`
    ErrorMessage    string                 `json:"errorMessage,omitempty"`
    ProcessingTime  time.Duration          `json:"processingTime"`
    ProcessedAt     time.Time              `json:"processedAt"`
    ProcessorID     string                 `json:"processorId"`
    Metadata        map[string]interface{} `json:"metadata,omitempty"`
}

// AckResponse represents a pending acknowledgment response
type AckResponse struct {
    CorrelationID string
    timeout       time.Duration
    createdAt     time.Time
}

// Wait for processing acknowledgment with timeout
func (r *AckResponse) WaitForAcknowledgment(ctx context.Context) (*ProcessingAcknowledgment, error)

// AcknowledgmentTracker manages application-level acknowledgments
type AcknowledgmentTracker struct {
    publisher        *MessagePublisher
    ackQueue         string
    pendingAcks      map[string]*AckResponse
    defaultTimeout   time.Duration
    cleanupInterval  time.Duration
    logger          *slog.Logger
}

// Send message and wait for acknowledgment
func (t *AcknowledgmentTracker) SendWithAck(
    ctx context.Context, 
    msg contracts.Message, 
    options ...PublishOption) (*AckResponse, error)

// Handle incoming acknowledgment
func (t *AcknowledgmentTracker) HandleAcknowledgment(ctx context.Context, ackMsg contracts.Message) error

// ProcessingAckHandler wraps handlers to send automatic acknowledgments
type ProcessingAckHandler struct {
    handler     MessageHandler
    publisher   *MessagePublisher
    logger      *slog.Logger
    processorID string
}

func NewProcessingAckHandler(
    handler MessageHandler, 
    publisher *MessagePublisher, 
    opts *ProcessingAckHandlerOptions) *ProcessingAckHandler
```

### Sync Mutation Journal

Entity-level mutation tracking for distributed synchronization.

```go
// EntityMutationType represents entity-level mutations for sync
type EntityMutationType string

const (
    EntityCreate EntityMutationType = "entity.create"
    EntityUpdate EntityMutationType = "entity.update"
    EntityDelete EntityMutationType = "entity.delete"
    EntityPatch  EntityMutationType = "entity.patch"
    EntityMerge  EntityMutationType = "entity.merge"
)

// SyncStatus represents the synchronization status of a mutation
type SyncStatus string

const (
    SyncStatusPending   SyncStatus = "pending"
    SyncStatusSynced    SyncStatus = "synced"
    SyncStatusFailed    SyncStatus = "failed"
    SyncStatusConflict  SyncStatus = "conflict"
    SyncStatusIgnored   SyncStatus = "ignored"
)

// EntityMutationRecord represents an entity-level mutation for synchronization
type EntityMutationRecord struct {
    ID             string                 `json:"id"`
    ServiceID      string                 `json:"serviceId"`
    EntityType     string                 `json:"entityType"`
    EntityID       string                 `json:"entityId"`
    EntityVersion  int64                  `json:"entityVersion"`
    MutationType   EntityMutationType     `json:"mutationType"`
    Timestamp      time.Time              `json:"timestamp"`
    CorrelationID  string                 `json:"correlationId"`
    CausationID    string                 `json:"causationId,omitempty"`
    Payload        json.RawMessage        `json:"payload"`
    BeforeState    json.RawMessage        `json:"beforeState,omitempty"`
    AfterState     json.RawMessage        `json:"afterState,omitempty"`
    SyncStatus     SyncStatus             `json:"syncStatus"`
    SyncedAt       *time.Time             `json:"syncedAt,omitempty"`
    SyncError      string                 `json:"syncError,omitempty"`
    ConflictsWith  []string               `json:"conflictsWith,omitempty"`
    Metadata       map[string]interface{} `json:"metadata,omitempty"`
}

// SyncMutationJournal extends the basic mutation journal with sync capabilities
type SyncMutationJournal interface {
    MutationJournal
    
    // RecordEntityMutation records an entity-level mutation
    RecordEntityMutation(ctx context.Context, record *EntityMutationRecord) error
    
    // GetEntityMutations retrieves mutations for a specific entity
    GetEntityMutations(ctx context.Context, entityType, entityID string) ([]*EntityMutationRecord, error)
    
    // GetCorrelatedMutations retrieves mutations with the same correlation ID
    GetCorrelatedMutations(ctx context.Context, correlationID string) ([]*EntityMutationRecord, error)
    
    // GetUnsyncedMutations retrieves mutations that haven't been synced
    GetUnsyncedMutations(ctx context.Context, limit int) ([]*EntityMutationRecord, error)
    
    // MarkAsSynced marks mutations as successfully synced
    MarkAsSynced(ctx context.Context, mutationIDs []string, syncedAt time.Time) error
    
    // MarkAsFailed marks mutations as sync failed
    MarkAsFailed(ctx context.Context, mutationIDs []string, syncError string) error
    
    // GetConflicts retrieves mutations that have sync conflicts
    GetConflicts(ctx context.Context) ([]*EntityMutationRecord, error)
    
    // ResolveConflict resolves a sync conflict
    ResolveConflict(ctx context.Context, mutationID string, resolution SyncStatus) error
}

// Create in-memory sync mutation journal
func NewInMemorySyncMutationJournal(
    baseOpts []InMemoryJournalOption, 
    syncOpts ...SyncJournalOption) *InMemorySyncMutationJournal

// Journal configuration options
func WithServiceID(serviceID string) SyncJournalOption
```

### Client Configuration

Enterprise features are configured through client options:

```go
// Enable TTL-based retry scheduling
func WithTTLRetry(policy ...RetryPolicy) ClientOption

// Enable acknowledgment tracking
func WithAcknowledgmentTracking(timeout ...time.Duration) ClientOption

// Enable sync mutation journal
func WithSyncMutationJournal(options ...SyncJournalOption) ClientOption

// Example: Client with all enterprise features
client, err := mmate.NewClientWithOptions(connectionString,
    mmate.WithTTLRetry(),                           // Enable TTL retry
    mmate.WithAcknowledgmentTracking(30*time.Second), // Enable ack tracking
    mmate.WithSyncMutationJournal(),                  // Enable sync journal
    mmate.WithServiceName("order-service"),
)
```

### Enterprise Client Methods

Additional methods available when enterprise features are enabled:

```go
// TTL Retry Scheduler access
func (c *Client) TTLRetryScheduler() *TTLRetryScheduler

// Acknowledgment Tracker access
func (c *Client) AcknowledgmentTracker() *AcknowledgmentTracker

// Sync Mutation Journal access
func (c *Client) SyncMutationJournal() SyncMutationJournal

// Send with acknowledgment tracking
func (c *Client) SendWithAck(ctx context.Context, msg contracts.Message, options ...PublishOption) (*AckResponse, error)

// Publish events with acknowledgment
func (c *Client) PublishEventWithAck(ctx context.Context, evt contracts.Event, options ...PublishOption) (*AckResponse, error)

// Publish commands with acknowledgment
func (c *Client) PublishCommandWithAck(ctx context.Context, cmd contracts.Command, options ...PublishOption) (*AckResponse, error)

// Record entity mutation
func (c *Client) RecordEntityMutation(ctx context.Context, record *EntityMutationRecord) error

// Get entity mutations
func (c *Client) GetEntityMutations(ctx context.Context, entityType, entityID string) ([]*EntityMutationRecord, error)

// Get unsynced mutations
func (c *Client) GetUnsyncedMutations(ctx context.Context, limit int) ([]*EntityMutationRecord, error)

// WorkflowCompensatedEvent published when compensation completes
type WorkflowCompensatedEvent struct {
    BaseEvent
    WorkflowID       string    `json:"workflowId"`
    WorkflowInstance string    `json:"workflowInstance"`
    FailedStage      string    `json:"failedStage"`
    CompensatedAt    time.Time `json:"compensatedAt"`
    CompensationStages []string `json:"compensationStages"`
}
```

### Workflow Engine

```go
type WorkflowEngine struct {
    publisher  Publisher
    subscriber Subscriber
    store      StateStore
    options    EngineOptions
}

// Constructor
func NewWorkflowEngine(publisher Publisher, subscriber Subscriber, 
    store StateStore, opts ...EngineOption) *WorkflowEngine

// Methods
func (e *WorkflowEngine) RegisterWorkflow(workflow Workflow) error
func (e *WorkflowEngine) StartWorkflow(ctx context.Context, workflowID string, 
    input interface{}) (string, error)
func (e *WorkflowEngine) GetWorkflowStatus(ctx context.Context, 
    instanceID string) (*WorkflowStatus, error)
func (e *WorkflowEngine) CancelWorkflow(ctx context.Context, instanceID string) error
```

## Bridge

### SyncAsyncBridge

The SyncAsyncBridge is integrated into the Client and accessed via `client.Bridge()`.

```go
// Access the bridge
bridge := client.Bridge()

// Main method
func (b *SyncAsyncBridge) SendAndWait(ctx context.Context, msg contracts.Message, 
    routingKey string, timeout time.Duration) (contracts.Reply, error)

// Bridge lifecycle (managed internally by client)
func (b *SyncAsyncBridge) Start(replyQueue string) error
func (b *SyncAsyncBridge) Stop() error
```

### ReplyStore Interface

```go
type ReplyStore interface {
    Store(correlationID string, reply Reply) error
    Get(correlationID string) (Reply, bool)
    Delete(correlationID string)
    SetChannel(correlationID string, ch chan<- Reply)
    GetChannel(correlationID string) (chan<- Reply, bool)
}
```

## Interceptors

### Interceptor Interface

```go
type Interceptor interface {
    Intercept(ctx context.Context, msg Message, next Handler) error
}

type Handler func(ctx context.Context, msg Message) error
```

### Pipeline

```go
type Pipeline struct {
    interceptors []Interceptor
}

// Constructor
func NewPipeline(interceptors ...Interceptor) *Pipeline

// Methods
func (p *Pipeline) Add(interceptor Interceptor) *Pipeline
func (p *Pipeline) Use(interceptor Interceptor) *Pipeline  // Alias for Add
func (p *Pipeline) Execute(ctx context.Context, msg Message, handler Handler) error
```

### Built-in Interceptors

```go
// Logging interceptor
type LoggingInterceptor struct {
    logger *slog.Logger
}

func NewLoggingInterceptor(logger *slog.Logger) *LoggingInterceptor

// Metrics interceptor
type MetricsInterceptor struct {
    collector MetricsCollector
}

func NewMetricsInterceptor(collector MetricsCollector) *MetricsInterceptor

// Validation interceptor
type ValidationInterceptor struct{}

func NewValidationInterceptor() *ValidationInterceptor

// Retry interceptor
type RetryInterceptor struct {
    policy RetryPolicy
}

func NewRetryInterceptor(policy RetryPolicy) *RetryInterceptor
```

### MetricsCollector Interface

```go
type MetricsCollector interface {
    RecordMessageSent(messageType string, exchange string, routingKey string)
    RecordMessageReceived(messageType string, queue string)
    RecordProcessingDuration(messageType string, duration time.Duration)
    RecordError(messageType string, errorType string)
}
```

## Reliability

### DLQHandler

```go
type DLQHandler struct {
    maxRetries        int
    dlqExchange       string
    dlqRoutingPrefix  string
    retryDelay        time.Duration
    logger            *slog.Logger
}

// Constructor
func NewDLQHandler(options ...DLQOption) *DLQHandler

// DLQ Options
type DLQOption func(*DLQHandler)

func WithMaxRetries(retries int) DLQOption
func WithDLQExchange(exchange string) DLQOption
func WithDLQRoutingPrefix(prefix string) DLQOption
func WithRetryDelay(delay time.Duration) DLQOption
func WithDLQLogger(logger *slog.Logger) DLQOption
```

## Monitoring

### Service-Scoped Monitoring Types

```go
// ServiceMonitor monitors a specific service's health and metrics
type ServiceMonitor struct {
    serviceName    string
    serviceQueue   string
    queueInspector *QueueInspector
}

// ServiceHealth represents the health status of a service
type ServiceHealth struct {
    Status           string                     `json:"status"`
    ServiceName      string                     `json:"serviceName"`
    QueueHealth      QueueHealth               `json:"queueHealth"`
    ConnectionHealth BasicConnectivityHealth   `json:"connectionHealth"`
    Timestamp        time.Time                 `json:"timestamp"`
}

// ServiceMetrics contains metrics for a service
type ServiceMetrics struct {
    ServiceName      string           `json:"serviceName"`
    QueueMetrics     QueueMetrics     `json:"queueMetrics"`
    MessageMetrics   MessageMetrics   `json:"messageMetrics"`
    ErrorRate        float64          `json:"errorRate"`
    Timestamp        time.Time        `json:"timestamp"`
}

// ConsumerStats tracks consumer-specific statistics
type ConsumerStats struct {
    ServiceName        string           `json:"serviceName"`
    ConsumerTag        string           `json:"consumerTag"`
    MessageRate        float64          `json:"messageRate"`
    ProcessingTime     time.Duration    `json:"processingTime"`
    LastMessageTime    time.Time        `json:"lastMessageTime"`
}

// QueueInspector provides queue inspection using AMQP
type QueueInspector struct {
    channelPool *rabbitmq.ChannelPool
    vhost       string
}

func (qi *QueueInspector) InspectQueue(ctx context.Context, queueName string) (*QueueInfo, error)

// BasicConnectivityHealth represents basic connection health
type BasicConnectivityHealth struct {
    Connected        bool      `json:"connected"`
    LastError        string    `json:"lastError,omitempty"`
    LastChecked      time.Time `json:"lastChecked"`
}
```

### MonitorClient (Deprecated)

```go
type MonitorClient struct {
    url    string
    client *http.Client
}

// Constructor
func NewMonitorClient(amqpURL string) (*MonitorClient, error)

// Methods
func (c *MonitorClient) GetOverview(ctx context.Context) (*Overview, error)
func (c *MonitorClient) ListQueues(ctx context.Context) ([]QueueInfo, error)
func (c *MonitorClient) GetQueue(ctx context.Context, name string) (*QueueInfo, error)
func (c *MonitorClient) ListExchanges(ctx context.Context) ([]ExchangeInfo, error)
func (c *MonitorClient) ListConnections(ctx context.Context) ([]ConnectionInfo, error)
func (c *MonitorClient) ListChannels(ctx context.Context) ([]ChannelInfo, error)
func (c *MonitorClient) Close() error
```

### Data Types

```go
type Overview struct {
    RabbitMQVersion string `json:"rabbitmq_version"`
    ManagementVersion string `json:"management_version"`
    Queues          []QueueInfo `json:"queues"`
    Exchanges       []ExchangeInfo `json:"exchanges"`
    Connections     []ConnectionInfo `json:"connections"`
    Channels        []ChannelInfo `json:"channels"`
    // ... additional fields
}

type QueueInfo struct {
    Name              string `json:"name"`
    VHost             string `json:"vhost"`
    Messages          int    `json:"messages"`
    MessagesReady     int    `json:"messages_ready"`
    MessagesUnacked   int    `json:"messages_unacknowledged"`
    Consumers         int    `json:"consumers"`
    ConsumerUtilisation float64 `json:"consumer_utilisation"`
    MessageStats      MessageStats `json:"message_stats"`
}

type ExchangeInfo struct {
    Name       string `json:"name"`
    Type       string `json:"type"`
    Durable    bool   `json:"durable"`
    AutoDelete bool   `json:"auto_delete"`
    Internal   bool   `json:"internal"`
}
```

## Health

### HealthChecker Interface

```go
type HealthChecker interface {
    Check(ctx context.Context) error
    Name() string
    Critical() bool
}
```

### Built-in Checkers

```go
// RabbitMQ health checker
type RabbitMQHealthChecker struct {
    client *MonitorClient
}

func NewRabbitMQHealthChecker(url string) (*RabbitMQHealthChecker, error)

// Custom health checker
type CustomHealthChecker struct {
    name     string
    checkFn  func(context.Context) error
    critical bool
}

func NewCustomHealthChecker(name string, checkFn func(context.Context) error, 
    critical bool) *CustomHealthChecker
```

### Health Service

```go
type HealthService struct {
    checkers []HealthChecker
}

// Constructor
func NewHealthService(checkers ...HealthChecker) *HealthService

// Methods
func (s *HealthService) Check(ctx context.Context) *HealthReport
func (s *HealthService) ServeHTTP(w http.ResponseWriter, r *http.Request)

// Data types
type HealthReport struct {
    Status    string                 `json:"status"`
    Timestamp time.Time              `json:"timestamp"`
    Checks    map[string]CheckResult `json:"checks"`
}

type CheckResult struct {
    Status   string        `json:"status"`
    Duration time.Duration `json:"duration"`
    Error    string        `json:"error,omitempty"`
}
```

## Common Types

### Context Extensions

```go
// Message context keys
type contextKey string

const (
    MessageIDKey     contextKey = "messageID"
    CorrelationIDKey contextKey = "correlationID"
    RetryCountKey    contextKey = "retryCount"
)

// Helper functions
func GetMessageID(ctx context.Context) string
func GetCorrelationID(ctx context.Context) string
func GetRetryCount(ctx context.Context) int
```

### Options Pattern

```go
// Common option types used throughout the framework
type Option[T any] func(*T)

// Apply options helper
func ApplyOptions[T any](target *T, opts ...Option[T]) {
    for _, opt := range opts {
        opt(target)
    }
}
```

### Retry Policy

```go
type RetryPolicy struct {
    MaxAttempts     int
    InitialDelay    time.Duration
    MaxDelay        time.Duration
    BackoffStrategy BackoffStrategy
}

type BackoffStrategy func(attempt int, initialDelay time.Duration) time.Duration

// Built-in strategies
var (
    LinearBackoff      BackoffStrategy
    ExponentialBackoff BackoffStrategy
    ConstantBackoff    BackoffStrategy
)
```