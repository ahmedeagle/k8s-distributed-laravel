# Laravel Kubernetes Health Checks Implementation

This document explains the health check implementation for the Laravel application running on Kubernetes.

## Overview

We've implemented three types of Kubernetes probes:

1. **Liveness Probe** - Determines if the container is alive
2. **Readiness Probe** - Determines if the container is ready to serve traffic
3. **Startup Probe** - Ensures the application has started before other probes begin

## Health Check Endpoints

### `/health/live` - Liveness Probe
- **Purpose**: Basic aliveness check
- **Response**: Simple JSON indicating the service is alive
- **Failure Action**: Kubernetes restarts the pod
- **Check Frequency**: Every 10 seconds (after 30s initial delay)

```json
{
  "status": "alive",
  "timestamp": "2025-10-03T10:30:00.000000Z",
  "service": "laravel-app"
}
```

### `/health/ready` - Readiness Probe
- **Purpose**: Comprehensive dependency check
- **Checks**: Database, Redis, Cache system
- **Response**: Detailed status of all dependencies
- **Failure Action**: Kubernetes stops sending traffic to the pod
- **Check Frequency**: Every 5 seconds (after 10s initial delay)

```json
{
  "status": "ready",
  "timestamp": "2025-10-03T10:30:00.000000Z",
  "service": "laravel-app",
  "checks": {
    "database": {
      "status": "healthy",
      "message": "Database connection successful"
    },
    "redis": {
      "status": "healthy",
      "message": "Redis connection successful"
    },
    "cache": {
      "status": "healthy",
      "message": "Cache system working"
    }
  }
}
```

### `/health/startup` - Startup Probe
- **Purpose**: Ensures application has fully initialized
- **Checks**: Config loading, Route loading
- **Response**: Startup status information
- **Failure Action**: Prevents other probes from running
- **Check Frequency**: Every 10 seconds (max 5 minutes startup time)

### `/health` - General Health Check
- **Purpose**: Comprehensive system information for monitoring
- **Includes**: System info, memory usage, uptime
- **Use Case**: External monitoring, debugging

## Kubernetes Probe Configuration

### Laravel Container Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 80
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /health/startup
    port: 80
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes max startup
```

### Nginx Container Probes

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /health/live
    port: 80
  initialDelaySeconds: 0
  periodSeconds: 2
  failureThreshold: 15  # 30 seconds max startup
```

## Probe Timing Guidelines

### Initial Delay Seconds
- **Startup Probe**: 0s (start immediately)
- **Readiness Probe**: 10s (wait for basic startup)
- **Liveness Probe**: 30s (wait for full initialization)

### Period Seconds (Check Frequency)
- **Startup Probe**: 10s (during startup phase)
- **Readiness Probe**: 5s (frequent checks for traffic routing)
- **Liveness Probe**: 10s (less frequent, avoid unnecessary restarts)

### Failure Thresholds
- **Startup Probe**: 30 failures (5 minutes max startup time)
- **Readiness Probe**: 3 failures (quick traffic removal)
- **Liveness Probe**: 3 failures (avoid premature restarts)

## Testing Health Checks

### Test Liveness Probe
```bash
kubectl exec -it <pod-name> -- curl http://localhost/health/live
```

### Test Readiness Probe
```bash
kubectl exec -it <pod-name> -- curl http://localhost/health/ready
```

### Test Startup Probe
```bash
kubectl exec -it <pod-name> -- curl http://localhost/health/startup
```

### View Probe Status
```bash
kubectl describe pod <pod-name>
```

### Check Pod Events
```bash
kubectl get events --field-selector involvedObject.name=<pod-name>
```

## Troubleshooting

### Common Issues

1. **Readiness Probe Failing**
   - Check database connectivity
   - Verify Redis connection
   - Check application logs

2. **Liveness Probe Failing**
   - Application might be hung or crashed
   - Check memory/CPU resources
   - Review application error logs

3. **Startup Probe Failing**
   - Application taking too long to start
   - Increase `failureThreshold` or `periodSeconds`
   - Check initialization dependencies

### Monitoring Commands

```bash
# Watch pod status
kubectl get pods -w

# Check probe history
kubectl describe pod <pod-name> | grep -A 10 "Conditions:"

# View container logs
kubectl logs <pod-name> -c laravel
kubectl logs <pod-name> -c nginx

# Check resource usage
kubectl top pod <pod-name>
```

## Best Practices

1. **Keep Liveness Probes Simple**: Only check basic application state
2. **Make Readiness Probes Comprehensive**: Check all dependencies
3. **Use Appropriate Timeouts**: Balance responsiveness vs. false positives
4. **Monitor Probe Metrics**: Track success/failure rates
5. **Test Probe Endpoints**: Ensure they work in all scenarios
6. **Consider Startup Time**: Set realistic startup probe thresholds

## Integration with Monitoring

These health check endpoints can be integrated with:
- Prometheus for metrics collection
- Grafana for visualization
- AlertManager for notifications
- External monitoring services

## Security Considerations

- Health check endpoints are publicly accessible
- Consider implementing authentication for detailed health info
- Limit sensitive information in health responses
- Use HTTPS in production environments