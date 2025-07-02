# CLI Monitor

The Mmate CLI monitor provides command-line access to monitoring and management functions. This tool is distributed as a standalone utility from the [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox).

## Installation

<table>
<tr>
<th>.NET</th>
<th>Go</th>
</tr>
<tr>
<td>

```bash
dotnet tool install -g MmateToolbox.Monitor
```

</td>
<td>

```bash
go install github.com/glimte/mmate-toolbox/cmd/monitor@latest
```

</td>
</tr>
</table>

> **Note**: These tools are maintained separately from the main Mmate implementation libraries. Visit the [mmate-toolbox repository](https://github.com/glimte/mmate-toolbox) for the latest releases and documentation.

## Usage

### Basic Commands

```bash
# Show overview
mmate monitor overview

# List all queues
mmate monitor queues

# Show specific queue details
mmate monitor queue orders

# List consumers
mmate monitor consumers

# Check system health
mmate monitor health
```

### Connection Options

```bash
# Connect to specific RabbitMQ instance
mmate monitor --url amqp://user:pass@host:5672 queues

# Use environment variable
export MMATE_AMQP_URL=amqp://localhost
mmate monitor queues
```

## Commands Reference

### `overview`
Display system overview including:
- RabbitMQ version
- Node information
- Message rates
- Resource usage

```bash
mmate monitor overview
```

### `queues`
List all queues with statistics:

```bash
# List all queues
mmate monitor queues

# Sort by message count
mmate monitor queues --sort depth

# Show only non-empty queues
mmate monitor queues --non-empty

# Filter by pattern
mmate monitor queues --filter "cmd.*"

# Limit results
mmate monitor queues --limit 10
```

Output columns:
- Queue name
- Messages ready
- Messages unacknowledged
- Total messages
- Consumers
- Message rates

### `queue <name>`
Show detailed information for a specific queue:

```bash
# Show queue details
mmate monitor queue orders

# Watch queue (refresh every 5 seconds)
mmate monitor queue orders --watch

# Custom refresh interval
mmate monitor queue orders --watch --interval 2s
```

### `consumers`
List all consumers:

```bash
# List all consumers
mmate monitor consumers

# Show consumers for specific queue
mmate monitor consumers --queue orders

# Show only active consumers
mmate monitor consumers --active
```

### `dlq`
Manage dead letter queues:

```bash
# Show DLQ statistics
mmate monitor dlq

# List messages in specific DLQ
mmate monitor dlq --queue dlq.orders

# Requeue messages from DLQ
mmate monitor dlq --requeue dlq.orders --count 100

# Requeue with confirmation
mmate monitor dlq --requeue dlq.orders --count 100 --confirm
```

### `health`
Check system health:

```bash
# Basic health check
mmate monitor health

# Detailed health check
mmate monitor health --detailed

# Check specific components
mmate monitor health --check rabbitmq
mmate monitor health --check consumers
mmate monitor health --check queues
```

### `perf`
Show performance metrics:

```bash
# Show current performance
mmate monitor perf

# Show performance for last hour
mmate monitor perf --period 1h

# Show performance for specific time range
mmate monitor perf --from "2024-01-15T10:00:00Z" --to "2024-01-15T11:00:00Z"
```

### `export`
Export monitoring data:

```bash
# Export to JSON
mmate monitor export --format json --output metrics.json

# Export to CSV
mmate monitor export --format csv --output queues.csv

# Export specific data
mmate monitor export --data queues --format json
mmate monitor export --data consumers --format csv
```

## Output Formats

### Table (default)
```
QUEUE                    READY   UNACKED   TOTAL   CONSUMERS   RATE
cmd.orders.create          125        10     135           3   25/s
evt.order.created           50         5      55           5   30/s
dlq.cmd.orders.create        3         0       3           0    0/s
```

### JSON
```bash
mmate monitor queues --format json
```

```json
[
  {
    "name": "cmd.orders.create",
    "messages_ready": 125,
    "messages_unacknowledged": 10,
    "messages_total": 135,
    "consumers": 3,
    "publish_rate": 25.0
  }
]
```

### CSV
```bash
mmate monitor queues --format csv
```

```csv
queue,ready,unacked,total,consumers,rate
cmd.orders.create,125,10,135,3,25.0
evt.order.created,50,5,55,5,30.0
```

## Filtering and Sorting

### Filtering
```bash
# Filter by pattern (supports wildcards)
mmate monitor queues --filter "evt.*"
mmate monitor queues --filter "*.orders.*"

# Filter by queue state
mmate monitor queues --non-empty
mmate monitor queues --idle
mmate monitor queues --active

# Filter by consumer count
mmate monitor queues --min-consumers 1
mmate monitor queues --max-consumers 5
```

### Sorting
```bash
# Sort options
mmate monitor queues --sort name     # Alphabetical
mmate monitor queues --sort depth    # By message count
mmate monitor queues --sort rate     # By publish rate
mmate monitor queues --sort consumers # By consumer count

# Reverse sort
mmate monitor queues --sort depth --reverse
```

## Watch Mode

Monitor changes in real-time:

```bash
# Watch all queues
mmate monitor queues --watch

# Watch specific queue
mmate monitor queue orders --watch

# Custom refresh interval
mmate monitor queues --watch --interval 2s

# Watch with alerts
mmate monitor queues --watch --alert-on "depth>1000"
```

## Scripting Support

### Exit Codes
- `0` - Success
- `1` - Connection error
- `2` - Authentication error
- `3` - Queue not found
- `4` - Operation failed

### Quiet Mode
```bash
# Suppress headers and formatting
mmate monitor queues --quiet

# Machine-readable output
mmate monitor queues --quiet --format json
```

### Examples

**Check if queue is empty:**
```bash
if [ $(mmate monitor queue orders --quiet --format json | jq .messages_total) -eq 0 ]; then
    echo "Queue is empty"
fi
```

**Alert on high queue depth:**
```bash
#!/bin/bash
THRESHOLD=1000
DEPTH=$(mmate monitor queue orders --quiet --format json | jq .messages_total)
if [ $DEPTH -gt $THRESHOLD ]; then
    send-alert "Queue depth critical: $DEPTH messages"
fi
```

**Export metrics to monitoring system:**
```bash
# JSON format for external processing
mmate monitor export --format json | jq '.queues[] | select(.messages > 100)'
```

## Configuration

### Configuration File
Create `~/.mmate/config.yaml`:

```yaml
connection:
  url: amqp://localhost
  timeout: 30s

monitor:
  default_format: table
  refresh_interval: 5s
  
alerts:
  high_queue_depth: 1000
  no_consumers: true
  high_error_rate: 0.05
```

### Environment Variables
```bash
export MMATE_AMQP_URL=amqp://user:pass@host:5672
export MMATE_FORMAT=json
export MMATE_REFRESH_INTERVAL=2s
```

## Best Practices

1. **Regular Monitoring**
   - Set up scheduled checks
   - Export metrics to monitoring systems
   - Configure alerts for anomalies

2. **Automation**
   - Use in CI/CD pipelines
   - Automate DLQ requeuing
   - Script regular health checks

3. **Performance**
   - Use filters to reduce data
   - Avoid frequent refreshes in production
   - Cache results when appropriate