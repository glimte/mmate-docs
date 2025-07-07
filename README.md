# Message Mate Documentation

Welcome to the unified documentation for Message Mate (Mmate) - a comprehensive messaging framework available for both .NET and Go.

Currently supports RabbitMQ and Amazon MQ, support for Azure ServiceBus and other message bus`s are planned.

## Overview

Mmate provides high-level abstractions for messaging patterns built on top of brokers like RabbitMQ, offering:
- Type-safe messaging with contracts
- Multiple messaging patterns (Pub/Sub, Request/Reply, etc.)
- Workflow orchestration with StageFlow
- Cross-cutting concerns via interceptors
- Built-in reliability patterns
- Cross-language compatibility between .NET and Go

## Quick Start

- **[Getting Started - .NET](getting-started/dotnet.md)** - Get started with .NET implementation
- **[Getting Started - Go](getting-started/go.md)** - Get started with Go implementation
- **[Architecture Overview](architecture.md)** - Understand the core concepts
- **[Quick Start Guide](quick-start.md)** - Quick introduction to Mmate

## Documentation Structure

### 📚 Core Documentation
- **[Architecture](architecture.md)** - System design and concepts
- **[Wire Format](wire-format.md)** - Cross-language message specification
- **[Patterns](patterns.md)** - Common messaging patterns with examples

### 📦 Components
- **[Contracts](components/contracts.md)** - Message types and interfaces
- **[Messaging](components/messaging.md)** - Publishers, subscribers, and handlers
- **[StageFlow](components/stageflow.md)** - Multi-stage workflow orchestration
- **[Bridge](components/bridge.md)** - Synchronous over asynchronous patterns
- **[Interceptors](components/interceptors.md)** - Cross-cutting concerns
- **[Schema](components/schema.md)** - Message validation and versioning
- **[Monitoring](components/monitoring.md)** - Health checks and metrics

### 🔷 Platform-Specific
- **[.NET Documentation](platform/dotnet/)** - .NET implementation details
  - [Examples](platform/dotnet/examples.md) - Complete .NET code examples
  - [API Reference](platform/dotnet/api-reference.md) - .NET API documentation
- **[Go Documentation](platform/go/)** - Go implementation details
  - [Examples](platform/go/examples.md) - Complete Go code examples
  - [API Reference](platform/go/api-reference.md) - Go API documentation

### 🔄 Migration
- **[Migration Guides](migration/)** - Moving between platforms
- **[.NET to Go](migration/dotnet-to-go.md)** - Migrate from .NET to Go
- **[Go to .NET](migration/go-to-dotnet.md)** - Migrate from Go to .NET

### 🛠️ Advanced Topics
- **[Complete Solutions](advanced/complete-solutions.md)** - Production-ready examples
- **[Response Tracking](advanced/response-tracking.md)** - Request-response lifecycle
- **[Reliability](advanced/reliability.md)** - Circuit breakers, retries, DLQ
- **[Performance](advanced/performance.md)** - Optimization and tuning
- **[Security](advanced/security.md)** - Authentication and authorization
- **[Testing](advanced/testing.md)** - Testing strategies
- **[Auto Acknowledgment](advanced/auto-acknowledgment.md)** - Message acknowledgment patterns
- **[StageFlow Workflows](advanced/stageflow-workflows.md)** - Advanced workflow patterns

### 🔧 Tools
- **[CLI Monitor](tools/cli-monitor.md)** - Command-line monitoring tool
- **[Tools Overview](tools/README.md)** - Available development and monitoring tools

## Platform Comparison

| Feature | .NET | Go |
|---------|------|-----|
| Minimum Version | .NET 6.0+ | Go 1.21+ |
| Dependency Injection | Built-in ASP.NET Core DI | Manual wiring |
| Async Model | async/await | Goroutines + Context |
| Configuration | appsettings.json | Environment variables |
| Testing | xUnit/NUnit | testing package |
| Package Manager | NuGet | Go modules |

## Getting Help

- 📖 Check the documentation for your platform
- 🐛 Report issues on the respective GitHub repository
- 💬 Community discussions on GitHub Discussions
- 📧 Commercial support available

## Contributing

See the contributing guide for your platform:
- [Contributing to .NET](https://github.com/mmate/mmate-dotnet/CONTRIBUTING.md)
- [Contributing to Go](https://github.com/glimte/mmate-go/CONTRIBUTING.md)
