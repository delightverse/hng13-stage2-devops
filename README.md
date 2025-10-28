# HNG Stage 2 - Blue/Green Deployment with NGINX

## Author
- **Name:** Ubah Delight Okechukwu
- **Slack:** @DelightU
- **Instance IP:** 13.217.227.104

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
git clone https://github.com/delightverse/hng13-stage2-devops.git
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
curl -i http://13.217.227.104:8080/version
```

## Deployment Information

This application is deployed on AWS EC2 and publicly accessible via:

- **NGINX Public Endpoint:** http://13.217.227.104:8080
- **Blue Container (Direct):** http://13.217.227.104:8081
- **Green Container (Direct):** http://13.217.227.104:8082

## Server Details

- **Platform:** AWS EC2
- **Instance Type:** t3.micro
- **OS:** Ubuntu 22.04 LTS
- **Region:** [us-east-1]
- **Docker Version:** [Docker version 28.5.1, build e180ab8]
- **Docker Compose Version:** [Docker Compose version v2.40.2]

## Trigger Tests

### Testing Failover
```bash
# Test to see if nginx is serving from blue
curl -v http://13.217.227.104:8080/version
# Wait 2 seconds
sleep 2

# Trigger chaos on Blue
curl -X POST "http://13.217.227.104:8081/chaos/start?mode=error"


# Test via NGINX (should route to Green)
curl -i http://13.217.227.104:8080/version | grep 'X-App-Pool'
# Wait 2 seconds
sleep 2

# Stop chaos
curl -X POST "http://13.217.227.104:8081/chaos/stop"

# Test to see if nginx is serving from blue
curl -v http://13.217.227.104:8081/version
```

### Verify No Client Errors
```bash
# Run multiple requests
for i in {1..20}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" http://13.217.227.104:8080/version
  curl -i http://13.217.227.104:8080/version | grep "X-App-Pool:"
done

# All should show 200
```

### Environment Variables (.env)

| Variable | Description | Example |
|----------|-------------|---------|
| `BLUE_IMAGE` | Blue container image | `https://hub.docker.com/r/yimikaade/wonderful:devops-stage-two` |
| `GREEN_IMAGE` | Green container image | `https://hub.docker.com/r/yimikaade/wonderful:devops-stage-two` |
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

1. Client sends request to `http://13.217.227.104:8080`
2. NGINX forwards to Blue (primary)
3. Blue responds with headers: `X-App-Pool: blue`
4. Client receives 200 OK

### Failover Scenario

1. Blue fails (chaos triggered or real failure)
2. NGINX detects failure (timeout or 5xx)
3. NGINX retries request to Green (backup)
4. Green responds with headers: 
   - `X-App-Pool: green`
   - `X-Release-Id: green-release-1.0.0`
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
curl -i http://13.217.227.104:8081/version | grep "X-App-Pool:"
```

### Headers Not Forwarding
```bash
# Test with verbose output
curl -v http://13.217.227.104:8080/version

# Should see X-App-Pool and X-Release-Id in response headers
```

## Cleanup
```bash
# Stop services
docker-compose down

# Remove volumes and networks
docker-compose down -v

# Remove images (optional)
docker rmi -f yimikaade/wonderful:devops-stage-two
```

## Project Structure
```
hng-stage2-blue-green/
├── docker-compose.yml        # Service orchestration
├── .env                      # Environment configuration
├── .env.example              # Example environment file
├── README.md                 # This files the step by step implementation and testing
└── nginx/                    # The nginx folder containing the files that will generate the conf template
    ├── nginx.conf.template   # NGINX configuration template
    └── entrypoint.sh         # Config generation script
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

**Built for HNG13 DevOps Track by @DelightU: Ubah Delight Okechukwu**