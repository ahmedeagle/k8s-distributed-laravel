# Health Check Probe Strategy Explanation

## Container Architecture
```
┌─────────────────┐    ┌──────────────────────┐
│   Nginx         │    │   Laravel (PHP-FPM)  │
│   Port: 80      │───▶│   Port: 9000         │
│   Protocol: HTTP│    │   Protocol: FastCGI  │
└─────────────────┘    └──────────────────────┘
```

## Probe Types by Container

### Laravel Container (PHP-FPM)
```yaml
# ✅ CORRECT - TCP Socket Check
livenessProbe:
  tcpSocket:
    port: 9000
  
# ❌ WRONG - HTTP Check  
livenessProbe:
  httpGet:
    path: /health/live
    port: 9000  # PHP-FPM doesn't understand HTTP!
```

**Why TCP?**
- PHP-FPM is a **process manager**, not a web server
- It only accepts **FastCGI protocol** connections
- TCP probe checks if the port is **open and accepting connections**
- This confirms PHP-FPM is running and ready to handle requests

### Nginx Container  
```yaml
# ✅ CORRECT - HTTP Check
livenessProbe:
  httpGet:
    path: /health/live
    port: 80
```

**Why HTTP?**
- Nginx **is** a web server that understands HTTP
- It can process the health check routes
- It forwards PHP requests to PHP-FPM via FastCGI
- This tests the **entire request flow**

## Request Flow for Health Checks

### HTTP Probe on Nginx (✅ Works)
```
1. Kubernetes → HTTP GET /health/live → Nginx:80
2. Nginx → FastCGI request → PHP-FPM:9000  
3. PHP-FPM → Processes Laravel health route
4. PHP-FPM → FastCGI response → Nginx
5. Nginx → HTTP response → Kubernetes
```

### TCP Probe on PHP-FPM (✅ Works)
```
1. Kubernetes → TCP connection attempt → PHP-FPM:9000
2. PHP-FPM → Accepts/Rejects connection
3. Connection status → Kubernetes
```

### HTTP Probe on PHP-FPM (❌ Fails)
```
1. Kubernetes → HTTP GET /health/live → PHP-FPM:9000
2. PHP-FPM → "I don't understand HTTP!" → Connection refused
3. Probe fails
```

## Summary

| Container | Port | Protocol | Probe Type | Reason |
|-----------|------|----------|------------|---------|
| Nginx | 80 | HTTP | httpGet | Web server, handles HTTP requests |
| PHP-FPM | 9000 | FastCGI | tcpSocket | Process manager, only FastCGI protocol |

The **tcpSocket** probe for PHP-FPM simply checks:
- "Is PHP-FPM process running?"
- "Is it accepting connections on port 9000?"
- "Can Nginx connect to it?"

This is the **correct and minimal** health check for PHP-FPM!