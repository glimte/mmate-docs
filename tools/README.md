# Mmate Tools

Mmate provides command-line tools for monitoring and managing your messaging system.

## Available Tools

### [CLI Monitor](cli-monitor.md)
Command-line tool for monitoring queues, consumers, and system health.

### [TUI Dashboard](tui-dashboard.md)
Interactive terminal UI dashboard for real-time monitoring.

## Installation

### CLI Monitor
```bash
# Go
go install github.com/mmate/mmate-go/cmd/monitor@latest

# .NET
dotnet tool install -g Mmate.Monitor
```

### TUI Dashboard
```bash
# Go
go install github.com/mmate/mmate-go/cmd/mmate-tui@latest

# .NET
dotnet tool install -g Mmate.Tui
```

## Quick Start

### Monitor queues
```bash
mmate monitor queues
```

### Start TUI dashboard
```bash
mmate-tui
```

### Check system health
```bash
mmate monitor health
```

## Common Use Cases

1. **Production Monitoring**
   - Real-time queue depth monitoring
   - Consumer health checks
   - Performance metrics

2. **Troubleshooting**
   - Dead letter queue inspection
   - Message flow tracing
   - Error analysis

3. **Operations**
   - Queue management
   - Message requeuing
   - System maintenance

## Configuration

Both tools support configuration via:
- Command-line flags
- Environment variables
- Configuration files

See individual tool documentation for details.