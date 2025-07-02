# TUI Dashboard

The Mmate TUI (Terminal User Interface) dashboard provides an interactive monitoring experience in your terminal. This tool is distributed as a standalone utility from the [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox).

## Installation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```bash
dotnet tool install -g MmateToolbox.Tui
```

</td>
<td>

```bash
go install github.com/glimte/mmate-toolbox/cmd/mmate-tui@latest
```

</td>
</tr>
</table>

> **Note**: These tools are maintained separately from the main Mmate implementation libraries. Visit the [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox) for the latest releases and documentation.

## Usage

### Basic Usage
```bash
# Start TUI dashboard
mmate-tui

# Connect to specific RabbitMQ instance
mmate-tui --url amqp://user:pass@host:5672

# Custom refresh interval
mmate-tui --refresh 2s
```

### Connection Options
```bash
# Connect to specific RabbitMQ instance
mmate-tui --url amqp://user:pass@host:5672

# Use environment variable
export MMATE_AMQP_URL=amqp://localhost
mmate-tui

# Connect with TLS
mmate-tui --url amqps://user:pass@host:5671 --tls
```

## Features

### Real-time Dashboard
- **Queue Monitoring**: Live statistics for all queues
- **Consumer Tracking**: Monitor active consumers and their performance
- **Message Flow**: Visualize message throughput and rates
- **Performance Metrics**: CPU, memory, and connection statistics
- **Alert Notifications**: Real-time alerts for critical conditions

### Interactive Navigation
- **Keyboard Controls**: Navigate using arrow keys and shortcuts
- **Filtering**: Filter queues by name, status, or message count
- **Sorting**: Sort by various metrics (depth, rate, consumers)
- **Drill-down Views**: Detailed information for specific queues

### Visual Elements
- **Real-time Graphs**: Sparkline charts for message rates
- **Color Coding**: Visual indicators for queue health
- **Progress Bars**: Visual representation of queue depths
- **Status Indicators**: Clear health status displays

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `↑`/`↓` | Navigate queue list |
| `Enter` | View queue details |
| `r` | Refresh data |
| `f` | Filter queues |
| `s` | Sort options |
| `d` | Toggle DLQ view |
| `c` | View consumers |
| `h` | Show help |
| `q` | Quit |

## Views

### Overview Dashboard
- System overview with key metrics
- RabbitMQ cluster information
- Total message counts and rates
- Connection and channel statistics

### Queue List View
```
┌─ Queues ────────────────────────────────────────────────────────────┐
│ NAME                    READY  UNACK  TOTAL  CONSUMERS  RATE        │
├─────────────────────────────────────────────────────────────────────┤
│ cmd.orders.create         125     10    135          3  ████▌ 25/s  │
│ evt.order.created          50      5     55          5  ██████ 30/s │
│ dlq.cmd.orders.create       3      0      3          0  ▄ 0/s       │
└─────────────────────────────────────────────────────────────────────┘
```

### Queue Detail View
```
┌─ Queue: cmd.orders.create ──────────────────────────────────────────┐
│ Ready:        125 messages                                          │
│ Unacked:       10 messages                                          │
│ Total:        135 messages                                          │
│ Consumers:      3 active                                            │
│ Publish Rate:  25.4/s  ████████████▌                               │
│ Consume Rate:  23.1/s  ███████████▌                                │
│                                                                     │
│ Recent Activity: ▂▃▅▇▆▄▂▁▃▅▇▆▄▂                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Consumer View
```
┌─ Consumers ─────────────────────────────────────────────────────────┐
│ TAG               QUEUE             CH  PREFETCH  RATE             │
├─────────────────────────────────────────────────────────────────────┤
│ consumer-001      cmd.orders.create  5        10  ████▌ 8.5/s      │
│ consumer-002      cmd.orders.create  7        10  ████▌ 8.1/s      │
│ consumer-003      cmd.orders.create  9        10  ████▌ 8.8/s      │
└─────────────────────────────────────────────────────────────────────┘
```

## Configuration

### Configuration File
Create `~/.mmate/tui-config.yaml`:

```yaml
connection:
  url: amqp://localhost
  timeout: 30s

display:
  refresh_interval: 5s
  theme: dark
  show_graphs: true
  
alerts:
  enabled: true
  sound: false
  high_queue_depth: 1000
  no_consumers: true
```

### Environment Variables
```bash
export MMATE_AMQP_URL=amqp://user:pass@host:5672
export MMATE_TUI_REFRESH=2s
export MMATE_TUI_THEME=dark
```

## Alerts and Notifications

The TUI dashboard can display real-time alerts:

- **High Queue Depth**: When queues exceed threshold
- **No Consumers**: When queues have messages but no consumers
- **Connection Issues**: When RabbitMQ connection problems occur
- **Performance Degradation**: When processing rates drop significantly

Alerts appear as overlays and can be configured to:
- Show desktop notifications (when supported)
- Log to file
- Send to external systems

## Troubleshooting

### Connection Issues
- Verify RabbitMQ is accessible: `curl -i http://localhost:15672/api/overview`
- Check credentials and permissions
- Ensure management plugin is enabled

### Display Issues
- Terminal must support 256 colors
- Minimum terminal size: 80x24
- Use `--no-color` flag for compatibility

### Performance
- Increase refresh interval for better performance: `--refresh 10s`
- Filter queues to reduce data load
- Use `--no-graphs` to disable visual charts

## Repository Information

Visit the [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox) for:
- Latest releases and installation instructions
- Source code and development information
- Issue tracking and feature requests
- Detailed configuration documentation