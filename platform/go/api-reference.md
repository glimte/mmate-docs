# Go API Reference

This document provides a complete API reference for the Mmate Go framework.

## Table of Contents

- [Contracts](#contracts)
- [Messaging](#messaging)
- [StageFlow](#stageflow)
- [Bridge](#bridge)
- [Interceptors](#interceptors)
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

## Messaging

### Publisher Interface

```go
type Publisher interface {
    PublishCommand(ctx context.Context, cmd Command) error
    PublishEvent(ctx context.Context, evt Event) error
    PublishQuery(ctx context.Context, qry Query) error
    PublishReply(ctx context.Context, reply Reply, replyTo string) error
}
```

### MessagePublisher

```go
type MessagePublisher struct {
    transport Transport
    options   PublisherOptions
}

// Constructor
func NewMessagePublisher(transport Transport, opts ...PublisherOption) *MessagePublisher

// Methods
func (p *MessagePublisher) PublishCommand(ctx context.Context, cmd Command) error
func (p *MessagePublisher) PublishEvent(ctx context.Context, evt Event) error
func (p *MessagePublisher) PublishQuery(ctx context.Context, qry Query) error
func (p *MessagePublisher) PublishReply(ctx context.Context, reply Reply, replyTo string) error

// Options
type PublisherOption func(*PublisherOptions)

func WithExchange(exchange string) PublisherOption
func WithConfirmMode(enabled bool) PublisherOption
func WithTimeout(timeout time.Duration) PublisherOption
```

### Subscriber Interface

```go
type Subscriber interface {
    Subscribe(ctx context.Context, queue, messageType string, handler Handler) error
    SubscribeWithOptions(ctx context.Context, queue, messageType string, 
        handler Handler, opts ...SubscriberOption) error
}

type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

### MessageSubscriber

```go
type MessageSubscriber struct {
    transport Transport
    options   SubscriberOptions
}

// Constructor
func NewMessageSubscriber(transport Transport, opts ...SubscriberOption) *MessageSubscriber

// Methods
func (s *MessageSubscriber) Subscribe(ctx context.Context, queue, messageType string, 
    handler Handler) error
func (s *MessageSubscriber) SubscribeWithOptions(ctx context.Context, queue, messageType string,
    handler Handler, opts ...SubscriberOption) error

// Options
type SubscriberOption func(*SubscriberOptions)

func WithPrefetchCount(count int) SubscriberOption
func WithRetryPolicy(policy RetryPolicy) SubscriberOption
func WithDeadLetterQueue(dlq string) SubscriberOption
func WithInterceptors(interceptors ...Interceptor) SubscriberOption
```

### Consumer Group

```go
type ConsumerGroup struct {
    name    string
    size    int
    options ConsumerGroupOptions
}

// Constructor
func NewConsumerGroup(name string, opts ...ConsumerGroupOption) *ConsumerGroup

// Methods
func (g *ConsumerGroup) Start(ctx context.Context, subscriber Subscriber, 
    queue string, handler Handler) error
func (g *ConsumerGroup) Stop() error
func (g *ConsumerGroup) Size() int
func (g *ConsumerGroup) ActiveWorkers() int

// Options
type ConsumerGroupOption func(*ConsumerGroupOptions)

func WithGroupSize(size int) ConsumerGroupOption
func WithPrefetchPerWorker(count int) ConsumerGroupOption
func WithWorkerRestartPolicy(policy RestartPolicy) ConsumerGroupOption
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

```go
type SyncAsyncBridge struct {
    publisher  Publisher
    subscriber Subscriber
    replyStore ReplyStore
    options    BridgeOptions
}

// Constructor
func NewSyncAsyncBridge(publisher Publisher, subscriber Subscriber,
    logger *slog.Logger, opts ...BridgeOption) (*SyncAsyncBridge, error)

// Methods
func (b *SyncAsyncBridge) SendAndWait(ctx context.Context, msg Message, 
    routingKey string, timeout time.Duration) (Reply, error)
func (b *SyncAsyncBridge) RequestCommand(ctx context.Context, cmd Command, 
    timeout time.Duration) (Reply, error)
func (b *SyncAsyncBridge) RequestQuery(ctx context.Context, qry Query, 
    timeout time.Duration) (Reply, error)
func (b *SyncAsyncBridge) Close() error

// Options
type BridgeOption func(*BridgeOptions)

func WithReplyTimeout(timeout time.Duration) BridgeOption
func WithReplyQueue(queue string) BridgeOption
func WithConcurrentRequests(limit int) BridgeOption
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
    registry *prometheus.Registry
}

func NewMetricsInterceptor(registry *prometheus.Registry) *MetricsInterceptor

// Validation interceptor
type ValidationInterceptor struct{}

func NewValidationInterceptor() *ValidationInterceptor

// Retry interceptor
type RetryInterceptor struct {
    policy RetryPolicy
}

func NewRetryInterceptor(policy RetryPolicy) *RetryInterceptor
```

## Monitoring

### MonitorClient

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