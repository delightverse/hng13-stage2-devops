# Implementation Decisions

## Overview

This document explains the technical decisions made during implementation of the Blue/Green deployment system.

## Key Design Choices

### 1. NGINX Upstream Configuration

**Decision:** Use primary/backup pattern instead of round-robin load balancing.

**Reasoning:**
- Task requires ALL traffic to Blue normally
- Green should only receive traffic when Blue fails
- The `backup` directive achieves this exactly

**Implementation:**
```nginx
upstream app_backend {
    server app_blue:3000 max_fails=1 fail_timeout=10s;
    server app_green:3000 backup max_fails=1 fail_timeout=10s;
}
```

### 2. Aggressive Timeout Values

**Decision:** Set tight timeouts (2-3 seconds).

**Reasoning:**
- Fast failure detection
- Task requires failover within 10 seconds
- Aggressive timeouts ensure failures are caught quickly
- Prevents slow clients from experiencing delays

**Configuration:**
```nginx
proxy_connect_timeout 2s;
proxy_send_timeout 3s;
proxy_read_timeout 3s;
```

### 3. Retry Policy

**Decision:** Retry on error, timeout, and all 5xx errors.

**Reasoning:**
- Covers all failure scenarios (chaos modes)
- Ensures zero client-visible errors
- Meets requirement: "client still receives 200"

**Configuration:**
```nginx
proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
proxy_next_upstream_tries 2;
```

### 4. Dynamic Configuration with envsubst

**Decision:** Use shell script + envsubst for config generation.

**Reasoning:**
- Need to determine backup pool dynamically
- If ACTIVE_POOL=blue → backup=green
- If ACTIVE_POOL=green → backup=blue
- envsubst handles variable substitution cleanly

**Alternative Considered:** Lua scripting in NGINX
**Rejected Because:** Adds complexity, envsubst is simpler

### 5. Health Check Configuration

**Decision:** Docker Compose health checks every 5 seconds.

**Reasoning:**
- Ensures containers are healthy before NGINX starts
- Prevents routing to unhealthy backends during startup
- 5-second interval balances responsiveness vs overhead

### 6. Header Forwarding

**Decision:** Explicitly forward X-App-Pool and X-Release-Id headers.

**Reasoning:**
- Task requires these headers in client response
- By default, NGINX might filter certain headers
- Explicit `proxy_pass_header` ensures forwarding

**Configuration:**
```nginx
proxy_pass_header X-App-Pool;
proxy_pass_header X-Release-Id;
```

### 7. Port Exposure

**Decision:** Expose Blue (8081) and Green (8082) directly.

**Reasoning:**
- Task requires grader to trigger chaos directly on containers
- Can't trigger chaos through NGINX (it would failover)
- Direct exposure allows testing specific container behavior

### 8. No Security Measures

**Decision:** No Docker Secrets, TLS, or authentication.

**Reasoning:**
- Task contains no sensitive data (all public config)
- Focus on core functionality over security theater
- Simplicity reduces failure risk with one-attempt limit
- Production would require proper secrets management

## Challenges Faced

### Challenge 1: Dynamic Backup Pool

**Problem:** NGINX config needs to know which pool is backup based on ACTIVE_POOL.

**Solution:** Shell script calculates backup pool and generates config using envsubst.

### Challenge 2: Ensuring Zero Client Errors

**Problem:** Client must never see 500 errors during failover.

**Solution:** 
- `proxy_next_upstream` retries within same request
- Aggressive timeouts catch failures fast
- Backup directive ensures Green is always ready

### Challenge 3: Header Forwarding

**Problem:** Initially headers weren't reaching client.

**Solution:** Added explicit `proxy_pass_header` directives.

## Testing Strategy

### Unit Tests (Individual Components)

1. Test Blue container directly
2. Test Green container directly
3. Test NGINX config generation
4. Test health check endpoints

### Integration Tests (Full System)

1. Normal operation (Blue active)
2. Failover on error mode
3. Failover on timeout mode
4. Multiple concurrent requests during failure
5. Recovery after chaos stops

### Load Tests
```bash
# 100 requests during chaos
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://13.217.227.104:8080/version
done | grep -v 200 | wc -l
# Should output: 0 (zero non-200 responses)
```

## Alternative Approaches Considered

### Option 1: Active-Active Load Balancing

**Description:** Both Blue and Green receive traffic simultaneously.

**Rejected Because:** 
- Task requires only one pool active at a time
- Doesn't meet "all traffic to Blue" requirement

### Option 2: Health Check-Based Routing

**Description:** NGINX actively polls /healthz and routes accordingly.

**Rejected Because:**
- Passive health checks (based on actual traffic) are simpler
- Active checks add overhead
- Task doesn't require active monitoring

### Option 3: Service Mesh (Istio/Linkerd)

**Description:** Use service mesh for advanced routing.

**Rejected Because:**
- Task explicitly prohibits service meshes
- Over-engineering for requirements
- Adds unnecessary complexity

## Performance Considerations

- **Fast Failover:** < 10s requirement met with 2-3s timeouts
- **Zero Downtime:** Backup pool always ready
- **Resource Efficiency:** Health checks every 5s (not too aggressive)
- **Network Overhead:** Minimal (single retry on failure)

## Monitoring & Observability

**Current Implementation:**
- Docker health checks
- NGINX access/error logs
- Container logs via docker-compose logs

**Production Improvements:**
- Prometheus metrics export
- Grafana dashboards
- ELK stack for centralized logging
- Alerting on failover events

## Compliance with Requirements
```bash
✅ Docker Compose orchestration
✅ NGINX upstream configuration
✅ Blue/Green exposed on 8081/8082
✅ NGINX on 8080
✅ Parameterized via .env
✅ Headers forwarded
✅ Automatic failover
✅ Zero non-200 responses
✅ Request completes within 10s
✅ Template-based config (envsubst)

❌ Not using: Kubernetes, Swarm, service mesh
❌ Not building: Using pre-built images only
```
## Lessons Learned

1. **Simplicity Wins:** Simple solutions are more reliable
2. **Test Early:** Failover testing caught config issues early
3. **Documentation Matters:** Clear docs prevent confusion
4. **One Attempt Pressure:** Forces careful planning and testing

## Future Enhancements

If this were a production system:
- Add TLS termination
- Implement circuit breaker pattern
- Add metrics and alerting
- Use external health check service
- Implement canary deployments
- Add A/B testing capabilities
- Automated rollback on metrics anomalies

## Conclusion

This implementation prioritizes:
1. **Correctness:** Meets all functional requirements
2. **Reliability:** Zero client errors during failover
3. **Simplicity:** Easy to understand and maintain
4. **Testability:** Clear verification steps

The design balances production-grade patterns with task-specific constraints.