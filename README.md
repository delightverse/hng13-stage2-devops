# HNG Stage 2 - Blue/Green Deployment with NGINX

## Author
- **Name:** [Your Full Name]
- **Slack:** @your-slack-username

## Overview

This project implements a Blue/Green deployment pattern using NGINX as a reverse proxy with automatic failover capabilities. Traffic is routed to a primary pool (Blue) with automatic failover to a backup pool (Green) when the primary experiences failures.

## Architecture
```
Client → NGINX (Port 8080) → Blue (8081) [Primary]
                           → Green (8082) [Backup]
```

## Features

- ✅ Automatic failover on Blue failure
- ✅ Zero client-visible errors (NGINX retries to Green)
- ✅ Health-based routing
- ✅ Parameterized configuration via .env
- ✅ Header forwarding (X-App-Pool, X-Release-Id)
- ✅ Fast failure detection (< 10s)

## Prerequisites

- Docker
- Docker Compose
- curl (for testing)

## Quick Start

### 1. Clone Repository
```bash
git clone https://github.com/YOUR-USERNAME/hng-stage2-blue-green.git
cd hng-stage2-blue-green
```

### 2. Configure Environment
```bash
cp .env.example .env
# Edit .env if needed (defaults are fine)
```

### 3. Start Services
```bash
docker-compose up -d
```

### 4. Verify Deployment
```bash
# Check all services are healthy
docker-compose ps

# Test through NGINX
curl http://localhost:8080/version
```

## Testing Failover

### Trigger Failure on Blue
```bash
# Induce error mode
curl -X POST "http://localhost:8081/chaos/start?mode=error"

# Wait 2 seconds
sleep 2

# Test - should now route to Green
curl http://localhost:8080/version
# Should show: X-App-Pool: green
```

### Verify No Client Errors
```bash
# Run multiple requests
for i in {1..20}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" http://localhost:8080/version
done

# All should show 200
```

### Stop Chaos
```bash
curl -X POST "http://localhost:8081/chaos/stop"
```

## Configuration

### Environment Variables (.env)

| Variable | Description | Example |
|----------|-------------|---------|
| `BLUE_IMAGE` | Blue container image | `ghcr.io/hngprojects/hng_stage_two_task:main` |
| `GREEN_IMAGE` | Green container image | `ghcr.io/hngprojects/hng_stage_two_task:main` |
| `ACTIVE_POOL` | Primary pool (blue/green) | `blue` |
| `RELEASE_ID_BLUE` | Blue release identifier | `blue-release-1.0.0` |
| `RELEASE_ID_GREEN` | Green release identifier | `green-release-1.0.0` |
| `PORT` | Application internal port | `3000` |

### Ports

| Service | Port | Purpose |
|---------|------|---------|
| NGINX | 8080 | Public entrypoint |
| Blue | 8081 | Direct access + chaos |
| Green | 8082 | Direct access + chaos |

## How It Works

### Normal Operation

1. Client sends request to `localhost:8080`
2. NGINX forwards to Blue (primary)
3. Blue responds with headers: `X-App-Pool: blue`
4. Client receives 200 OK

### Failover Scenario

1. Blue fails (chaos triggered or real failure)
2. NGINX detects failure (timeout or 5xx)
3. NGINX retries request to Green (backup)
4. Green responds with headers: `X-App-Pool: green`
5. Client receives 200 OK (never sees Blue's failure)

### NGINX Configuration

Key settings:
- `max_fails=1` - Mark down after 1 failure
- `fail_timeout=10s` - Retry after 10 seconds
- `proxy_next_upstream` - Retry on error/timeout/5xx
- `backup` - Green only receives traffic when Blue is down
- Aggressive timeouts (2-3s) for fast failover

## Troubleshooting

### Services Not Starting
```bash
# Check logs
docker-compose logs

# Check if ports are already in use
lsof -i :8080
lsof -i :8081
lsof -i :8082
```

### Failover Not Working
```bash
# Check NGINX config
docker-compose exec nginx cat /etc/nginx/conf.d/default.conf

# Check NGINX error logs
docker-compose logs nginx

# Verify Blue is actually failing
curl http://localhost:8081/version
```

### Headers Not Forwarding
```bash
# Test with verbose output
curl -v http://localhost:8080/version

# Should see X-App-Pool and X-Release-Id in response headers
```

## Cleanup
```bash
# Stop services
docker-compose down

# Remove volumes and networks
docker-compose down -v

# Remove images (optional)
docker rmi ghcr.io/hngprojects/hng_stage_two_task:main
```

## Project Structure
```
hng-stage2-blue-green/
├── docker-compose.yml          # Service orchestration
├── .env                        # Environment configuration
├── .env.example               # Example environment file
├── README.md                  # This file
├── DECISION.md               # Implementation decisions
└── nginx/
    ├── nginx.conf.template   # NGINX configuration template
    └── entrypoint.sh        # Config generation script
```

## Technical Decisions

See [DECISION.md](DECISION.md) for detailed explanations of implementation choices.

## References

- [NGINX Upstream Documentation](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [NGINX Load Balancing](https://nginx.org/en/docs/http/load_balancing.html)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Blue/Green Deployment Pattern](https://martinfowler.com/bliki/BlueGreenDeployment.html)

## License

Educational project for HNG Internship Stage 2.

---

**Built with ❤️ for HNG13 DevOps Track**