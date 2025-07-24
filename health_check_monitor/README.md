# Health Check Monitor

A Docker container that monitors RabbitMQ health check messages and sends Discord alerts when health checks are missed.

## Features

- Consumes health check messages from RabbitMQ queue
- Monitors for missing health checks with configurable timeout
- Sends Discord webhook alerts when timeouts occur
- Dockerized for easy deployment
- Includes RabbitMQ instance in docker-compose setup

## Configuration

Copy `.env.example` to `.env` and configure:

```bash
cp .env.example .env
```

Required environment variables:

- `DISCORD_WEBHOOK_URL`: Discord webhook URL for alerts
- `RABBITMQ_URL`: RabbitMQ connection string

Optional configuration variables:

- `HEALTH_CHECK_QUEUE`: Queue name for health check messages (default: `health_checks`)
- `RABBITMQ_EXCHANGE_NAME`: Exchange name for health check messages (default: `wbor_failsafe_events`)  
- `RABBITMQ_HEALTHCHECK_ROUTING_KEY`: Routing key for health check messages (default: `health.failsafe-status`, the same as the main notifier)
- `CHECK_INTERVAL_SECONDS`: How often to check for timeouts (default: `300`)
- `TIMEOUT_THRESHOLD_SECONDS`: Timeout threshold before alerting (default: `600`)

**RabbitMQ Message Routing:**
The consumer automatically binds the health check queue to the specified exchange with the routing key. This allows it to receive health check messages published by the main failsafe notifier service.

## Usage

### Using Makefile (Recommended)

The included Makefile provides convenient commands for container management:

```bash
# Quick rebuild and run (stops existing container, builds, runs, follows logs)
make

# Build the image
make build

# Run the container (creates logs directory, uses .env file)
make run

# Follow container logs
make logsf

# Check container health status
make health

# Execute shell in running container
make exec

# Stop and remove container
make stop

# Clean up (stop container and remove image)
make clean
```

**Environment Configuration:**

- The Makefile uses `.env` file for configuration
- Supports both Docker and Podman via `DOCKER_TOOL` environment variable
- Creates local `logs/` directory for persistent logging

### Using Docker Compose

```bash
# Start the services
docker-compose up -d

# View logs
docker-compose logs -f health-check-monitor

# Stop the services
docker-compose down
```

**Environment Variable Precedence (highest to lowest):**

1. Command line variables (`docker-compose run -e VAR=value`)
2. Shell environment variables
3. **`.env` file** (takes precedence over docker-compose.yml)
4. `environment` section in docker-compose.yml
5. Dockerfile ENV statements

### Using Docker directly

```bash
# Build the image
docker build -t health-check-monitor .

# Run the container
docker run -d \
  --name health-check-monitor \
  -e DISCORD_WEBHOOK_URL="your_webhook_url" \
  -e RABBITMQ_URL="amqp://guest:guest@localhost:5672/" \
  health-check-monitor
```

### Using with External RabbitMQ Container

If you already have a RabbitMQ container running, you can use the health check monitor standalone:

1. **Remove RabbitMQ from docker-compose.yml** or create a new compose file:

    ```yaml
    services:
      health-check-monitor:
        build: .
        restart: unless-stopped
        environment:
          - RABBITMQ_URL=amqp://guest:guest@your-rabbitmq-host:5672/
          - HEALTH_CHECK_QUEUE=health_checks
          - DISCORD_WEBHOOK_URL=${DISCORD_WEBHOOK_URL}
          - CHECK_INTERVAL_SECONDS=300
          - TIMEOUT_THRESHOLD_SECONDS=600
        networks:
          - your-existing-network

    networks:
      your-existing-network:
        external: true
    ```

2. **Update the RABBITMQ_URL** in your `.env` file to point to your existing RabbitMQ instance:

    ```bash
    # Point to existing RabbitMQ container
    RABBITMQ_URL=amqp://username:password@rabbitmq-container-name:5672/

    # Or point to external RabbitMQ server
    RABBITMQ_URL=amqp://username:password@rabbitmq.example.com:5672/
    ```

3. **Ensure the health check queue exists** in your RabbitMQ instance, or the consumer will create it automatically when it starts.

4. **Connect to the same Docker network** if your RabbitMQ container is in a custom network:

    ```bash
    # Add your monitor to the existing network
    docker network connect your-rabbitmq-network health-check-monitor
    ```

## Health Check Message Format

The consumer expects JSON messages in the following format:

```json
{
  "timestamp": "2025-01-24T10:30:00Z",
  "service": "your-service-name",
  "status": "healthy"
}
```

## Monitoring

- RabbitMQ Management UI: <http://localhost:15672> (credentials guest/guest unless changed)
- Container logs show health check activity and alerts
