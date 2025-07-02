# Mmate Tools

Mmate provides external command-line tools for monitoring and managing your messaging system. These tools are distributed from the [mmate-toolbox](https://github.com/glimte/mmate-toolbox) repository as standalone utilities.

## Available Tools

### [CLI Monitor](cli-monitor.md)
Command-line tool for monitoring queues, consumers, and system health.

### [TUI Dashboard](tui-dashboard.md)
Interactive terminal UI dashboard for real-time monitoring.

## Installation

All tools are distributed from the external [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox).

### CLI Monitor
```bash
# Go
go install github.com/glimte/mmate-toolbox/cmd/monitor@latest

# .NET
dotnet tool install -g MmateToolbox.Monitor
```

### TUI Dashboard
```bash
# Go
go install github.com/glimte/mmate-toolbox/cmd/mmate-tui@latest

# .NET
dotnet tool install -g MmateToolbox.Tui
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

## Repository Information

The Mmate CLI tools are maintained in a separate repository to provide standalone utilities that can be used independently of the main Mmate implementation libraries:

- **Repository**: [github.com/glimte/mmate-toolbox](https://github.com/glimte/mmate-toolbox)
- **Documentation**: Available in the toolbox repository
- **Releases**: Check the toolbox repository for latest releases
- **Issues**: Report tool-specific issues in the toolbox repository

This separation allows the tools to be updated independently and provides a cleaner installation experience for users who only need monitoring capabilities.