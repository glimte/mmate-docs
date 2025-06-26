# Getting Started with Mmate

This guide will help you get started with Mmate on your platform of choice.

## Choose Your Platform

<table>
<tr>
<td width="50%">

### 🔷 [.NET Getting Started](../dotnet/quick-start.md)

Perfect if you're using:
- C# / F# / VB.NET
- ASP.NET Core
- .NET 6.0 or later
- Visual Studio / Rider

**Quick Install:**
```bash
dotnet add package Mmate
dotnet add package Mmate.StageFlow
```

</td>
<td width="50%">

### 🔶 [Go Getting Started](../go/getting-started.md)

Perfect if you're using:
- Go 1.21 or later
- Microservices architecture
- Cloud-native applications
- VS Code / GoLand

**Quick Install:**
```bash
go get github.com/glimte/mmate-go
```

</td>
</tr>
</table>

## Prerequisites (Both Platforms)

### 1. RabbitMQ

You'll need RabbitMQ 3.8 or later. Choose one:

**Using Docker (Recommended for development):**
```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:3-management
```

**Using Docker Compose:**
```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

volumes:
  rabbitmq_data:
```

**Local Installation:**
- macOS: `brew install rabbitmq`
- Ubuntu: `sudo apt-get install rabbitmq-server`
- Windows: Download from [RabbitMQ website](https://www.rabbitmq.com/download.html)

### 2. Verify Installation

Access RabbitMQ Management UI:
- URL: http://localhost:15672
- Default credentials: guest/guest

## Quick Examples

### Hello World - Publisher

<table>
<tr>
<td>

**.NET**
```csharp
using Mmate;

// Configure
var publisher = new MessagePublisher(
    "amqp://localhost");

// Publish event
await publisher.PublishEventAsync(
    new HelloEvent { 
        Message = "Hello, Mmate!" 
    });
```

</td>
<td>

**Go**
```go
import "github.com/glimte/mmate-go"

// Configure
publisher := mmate.NewPublisher(
    "amqp://localhost")

// Publish event
err := publisher.PublishEvent(ctx, 
    &HelloEvent{
        Message: "Hello, Mmate!",
    })
```

</td>
</tr>
</table>

### Hello World - Consumer

<table>
<tr>
<td>

**.NET**
```csharp
// Define handler
public class HelloHandler : 
    IMessageHandler<HelloEvent>
{
    public async Task HandleAsync(
        HelloEvent message)
    {
        Console.WriteLine(
            $"Received: {message.Message}");
    }
}

// Subscribe
await subscriber.SubscribeAsync<HelloEvent>(
    "hello.queue",
    new HelloHandler());
```

</td>
<td>

**Go**
```go
// Define handler
type HelloHandler struct{}

func (h *HelloHandler) Handle(
    ctx context.Context, 
    msg contracts.Message) error {
    
    event := msg.(*HelloEvent)
    fmt.Printf("Received: %s\n", 
        event.Message)
    return nil
}

// Subscribe
err := subscriber.Subscribe(ctx,
    "hello.queue",
    "HelloEvent",
    &HelloHandler{})
```

</td>
</tr>
</table>

## Core Concepts

### 1. Messages

All messages in Mmate fall into one of four categories:

- **Commands** - Instructions to do something
- **Events** - Notifications that something happened
- **Queries** - Requests for information
- **Replies** - Responses to commands or queries

### 2. Message Flow

```
Publisher → Exchange → Queue → Consumer → Handler
```

### 3. Key Components

- **Publisher** - Sends messages
- **Subscriber** - Receives messages
- **Handler** - Processes messages
- **Interceptor** - Cross-cutting concerns
- **Bridge** - Sync-over-async patterns
- **StageFlow** - Workflow orchestration

## Next Steps

### By Feature

1. **Basic Messaging**
   - [Message Patterns](../patterns/README.md)
   - [Contracts](../components/contracts/README.md)
   - [Publishing Messages](../components/messaging/README.md#publishing)
   - [Consuming Messages](../components/messaging/README.md#consuming)

2. **Advanced Patterns**
   - [Request/Reply with Bridge](../components/bridge/README.md)
   - [Workflows with StageFlow](../components/stageflow/README.md)
   - [Consumer Groups](../patterns/consumer-groups.md)

3. **Production Features**
   - [Interceptors](../components/interceptors/README.md)
   - [Reliability Patterns](../advanced/reliability.md)
   - [Monitoring](../components/monitoring/README.md)
   - [Performance Tuning](../advanced/performance.md)

### By Learning Path

**New to Message-Driven Architecture?**
1. Read [Architecture Overview](../architecture/README.md)
2. Try the [Hello World Example](#hello-world---publisher)
3. Learn about [Message Patterns](../patterns/README.md)

**Migrating from Another System?**
1. Check [Migration Guides](../migration/README.md)
2. Review [API References](../README.md#api-references)
3. Study [Examples](../README.md#examples)

**Building Production Systems?**
1. Implement [Health Monitoring](../components/monitoring/README.md)
2. Configure [Reliability Patterns](../advanced/reliability.md)
3. Plan [Deployment Strategy](../advanced/deployment.md)

## Common Tasks

### Setting Up Development Environment

1. Install platform SDK (.NET SDK or Go)
2. Install RabbitMQ (see above)
3. Install IDE/Editor:
   - .NET: Visual Studio, Rider, VS Code + C# extension
   - Go: VS Code + Go extension, GoLand, Vim + vim-go
4. Clone examples repository
5. Run tests to verify setup

### Creating Your First Service

1. Create new project
2. Add Mmate package/module
3. Define message contracts
4. Implement handlers
5. Configure publisher/subscriber
6. Run and test

### Debugging Tips

- Enable debug logging
- Use RabbitMQ Management UI to inspect queues
- Check correlation IDs for request tracking
- Monitor dead letter queues
- Use interceptors for detailed logging

## Getting Help

- 📚 Read platform-specific documentation
- 💻 Check example projects
- 🐛 Search existing issues
- ❓ Ask in discussions
- 📧 Contact support (commercial users)

## Ready to Start?

Choose your platform and dive in:

- **[.NET Detailed Guide →](../dotnet/quick-start.md)**
- **[Go Detailed Guide →](../go/quick-start.md)**

Or explore:
- **[Example Projects →](../README.md#examples)**
- **[API Documentation →](../README.md#api-references)**