# Spring Boot Microservice Runbook: 

## 1. Overview
This runbook provides operational procedures for Spring Boot microservices in production environments. It covers deployment, monitoring, troubleshooting, and maintenance procedures.

## 2. Service Information

### 2.1 Basic Details
- **Service Name**: `[Service Name]`
- **Version**: `[Current Version]`
- **Port**: `[Service Port]`
- **Environment**: `[Prod/Staging/Dev]`
- **Repository**: `[Git Repository URL]`
- **Team Contact**: `[Team Email/Slack Channel]`

### 2.2 Dependencies
```
[Upstream Services]:
- Service A: [Version/Endpoint]
- Service B: [Version/Endpoint]

[Downstream Services]:
- Database: [Type/Cluster]
- Message Broker: [Type/Cluster]
- External APIs: [List]
```

## 3. Deployment Procedures

### 3.1 Pre-Deployment Checklist
- [ ] Code review completed
- [ ] All tests passed (unit, integration, security)
- [ ] Performance tests completed
- [ ] Database migrations ready (if applicable)
- [ ] Configuration updated for target environment
- [ ] Dependencies verified
- [ ] Rollback plan documented
- [ ] Stakeholders notified

### 3.2 Deployment Steps

#### Blue-Green Deployment
```bash
# Deploy new version (green environment)
./deploy.sh --environment green --version [NEW_VERSION]

# Run health checks
curl -X GET http://green-instance:port/actuator/health

# Switch traffic (update load balancer)
./switch-traffic.sh --to green

# Monitor for 15 minutes
./monitor-metrics.sh --duration 15m

# If stable, decommission old version (blue)
./shutdown.sh --environment blue
```

#### Canary Deployment
```bash
# Deploy to 10% of instances
./deploy-canary.sh --percentage 10 --version [NEW_VERSION]

# Monitor canary metrics
./monitor-canary.sh --duration 30m

# If metrics acceptable, roll out to 50%
./expand-deployment.sh --percentage 50

# Monitor again, then full deployment
./expand-deployment.sh --percentage 100
```

### 3.3 Rollback Procedure
```bash
# Immediately if critical issues detected
./rollback.sh --to-version [PREVIOUS_VERSION]

# Steps:
# 1. Stop new version
# 2. Restore previous version
# 3. Verify health
# 4. Notify team
```

## 4. Monitoring & Alerts

### 4.1 Health Checks
```bash
# Basic health
curl http://localhost:8080/actuator/health

# Detailed health with components
curl http://localhost:8080/actuator/health/{component}

# Custom health endpoint
curl http://localhost:8080/api/v1/health
```

### 4.2 Key Metrics to Monitor
| Metric | Threshold | Alert Channel | Response Time |
|--------|-----------|---------------|---------------|
| CPU Usage | > 80% for 5min | PagerDuty | 15 minutes |
| Memory Usage | > 85% for 5min | Slack | 15 minutes |
| Error Rate | > 5% for 2min | PagerDuty | 5 minutes |
| Response Time (p95) | > 2s for 5min | Slack | 30 minutes |
| JVM GC Pauses | > 1s frequency | Slack | 60 minutes |

### 4.3 Log Aggregation
```bash
# Search recent errors
kubectl logs deployment/[service-name] --tail=100 | grep ERROR

# Follow logs
kubectl logs deployment/[service-name] -f

# Query centralized logs (ELK/Splunk)
# Example: Errors in last hour
index=springboot-prod ERROR | timechart count
```

## 5. Troubleshooting Guide

### 5.1 Service Won't Start
**Symptoms**: Application fails to start, port already in use, dependency errors

**Diagnosis Steps**:
1. Check logs: `journalctl -u [service-name]`
2. Verify port availability: `netstat -tulpn | grep :8080`
3. Check disk space: `df -h`
4. Validate configuration: `java -jar app.jar --spring.config.location=...`

**Resolution**:
```bash
# Kill process on occupied port
sudo lsof -ti:8080 | xargs kill -9

# Clear temp files
rm -rf /tmp/*

# Restart with debug
java -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=5005,suspend=n \
     -jar application.jar
```

### 5.2 High Memory Usage
**Symptoms**: OOM errors, slow response, frequent GC

**Diagnosis**:
```bash
# Check JVM memory
jstat -gc [pid] 1000

# Heap dump (if enabled)
jmap -dump:live,format=b,file=heap.hprof [pid]

# Analyze with jvisualvm or Eclipse MAT
```

**Resolution**:
1. Increase heap: `-Xmx4g -Xms4g`
2. Adjust GC: `-XX:+UseG1GC -XX:MaxGCPauseMillis=200`
3. Restart with new settings
4. Profile for memory leaks

### 5.3 Database Connection Issues
**Symptoms**: Connection pool exhausted, slow queries, timeout errors

**Diagnosis**:
```bash
# Check connection pool metrics
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active

# Database connections
SELECT * FROM pg_stat_activity WHERE application_name = '[app-name]';
```

**Resolution**:
```yaml
# Update application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### 5.4 Circuit Breaker Tripped
**Symptoms**: Fail-fast errors, degraded functionality

**Diagnosis**:
```bash
# Check circuit breaker state
curl http://localhost:8080/actuator/health
curl http://localhost:8080/actuator/hystrix.stream
```

**Resolution**:
1. Identify failing dependency
2. Check dependent service health
3. Temporarily increase timeout:
```yaml
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: false
      circuitBreaker:
        requestVolumeThreshold: 20
```

## 6. Maintenance Procedures

### 6.1 Database Migration
```bash
# Dry run
./migrate.sh --env prod --dry-run

# Execute migration
./migrate.sh --env prod --confirm

# Verify
./verify-migration.sh --version [migration-version]
```

### 6.2 Certificate Renewal
```bash
# Check expiry
openssl x509 -in cert.pem -noout -dates

# Deploy new certificate
kubectl create secret tls app-tls --cert=new-cert.pem --key=new-key.pem

# Rolling restart
kubectl rollout restart deployment/[service-name]
```

### 6.3 Configuration Updates
```bash
# Update config map
kubectl create configmap app-config --from-file=application.yml -o yaml --dry-run | \
kubectl apply -f -

# Refresh without restart (if Spring Cloud Config)
POST http://localhost:8080/actuator/refresh
```

## 7. Disaster Recovery

### 7.1 Service Restoration
```bash
# Scale up
kubectl scale deployment/[service-name] --replicas=3

# Drain and restart problematic nodes
kubectl drain [node-name] --ignore-daemonsets

# Restore from backup (if needed)
./restore-backup.sh --type database --timestamp [backup-time]
```

### 7.2 Data Corruption Recovery
1. Stop service: `kubectl scale deployment/[service-name] --replicas=0`
2. Restore last known good backup
3. Run data integrity checks
4. Start service with reduced capacity
5. Monitor closely

## 8. Performance Tuning

### 8.1 JVM Optimization
```bash
# Production recommended settings
java -server \
     -Xms2g -Xmx2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:+UseStringDeduplication \
     -jar application.jar
```

### 8.2 Spring Boot Tuning
```yaml
# application-prod.yml
server:
  tomcat:
    max-threads: 200
    min-spare-threads: 10
    max-connections: 10000
    connection-timeout: 5000
    
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

## 9. Security Procedures

### 9.1 Vulnerability Scan
```bash
# Dependency check
./mvnw dependency-check:check
# or
./gradlew dependencyCheckAnalyze

# Container scanning
trivy image [image-name]:[tag]

# Update dependencies
./mvnw versions:use-latest-releases
```

### 9.2 Secret Rotation
1. Generate new secrets
2. Update Kubernetes secrets or vault
3. Rolling restart of pods
4. Verify old secrets are invalidated

## 10. Appendix

### 10.1 Useful Commands
```bash
# Get pod status
kubectl get pods -l app=[service-name]

# Enter container
kubectl exec -it [pod-name] -- /bin/bash

# Describe service issues
kubectl describe pod [pod-name]

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 10.2 Contact Escalation
1. **Level 1**: On-call Engineer (24/7)
2. **Level 2**: Senior Engineer (Business hours)
3. **Level 3**: Architecture Team
4. **Level 4**: Vendor Support (if applicable)

### 10.3 Post-Incident Report Template
```
# Incident Report: [Date]
## Summary
## Timeline
## Root Cause
## Impact
## Resolution
## Action Items
## Prevention Measures
```

---

**Last Updated**: [Date]
**Maintainer**: [Team Name]
**Review Schedule**: Bi-weekly