# Spring Boot Microservice Runbook

**Service Name**: `user-service`  
**Version**: `1.2.0`  
**Last Updated**: January 17, 2026  
**Owner**: Platform Engineering Team  

---

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Deployment](#deployment)
4. [Configuration](#configuration)
5. [Health Checks & Monitoring](#health-checks--monitoring)
6. [Logging](#logging)
7. [Scaling](#scaling)
8. [Troubleshooting](#troubleshooting)
9. [Rollback Procedure](#rollback-procedure)
10. [Contact Information](#contact-information)

---

## Overview
The `user-service` manages user authentication, profile data, and role-based access control. It exposes REST APIs for user registration, login, profile updates, and permission management.

**Key Features**:
- JWT-based authentication
- PostgreSQL database integration
- Redis caching for session management
- Integration with `notification-service` via RabbitMQ

---

## Prerequisites
### Dependencies
- **Java 17+**
- **Maven 3.8+**
- **Docker 24.0+** (for containerized deployment)
- **Kubernetes 1.25+** (if deploying to K8s)

### External Services
| Service          | Purpose                     | Required? |
|------------------|-----------------------------|-----------|
| PostgreSQL 14    | Primary user data storage   | Yes       |
| Redis 7          | Session cache               | Yes       |
| RabbitMQ 3.11    | Async notifications         | Optional  |
| Keycloak 22      | Identity provider           | Yes       |

---

## Deployment
### Local Development
```bash
# Clone repository
git clone https://github.com/your-org/user-service.git
cd user-service

# Build & run
./mvnw clean spring-boot:run
```

### Docker
```bash
# Build image
docker build -t user-service:1.2.0 .

# Run container
docker run -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_URL=jdbc:postgresql://db-host:5432/users \
  user-service:1.2.0
```

### Kubernetes
```yaml
# Apply manifests
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# Verify deployment
kubectl get pods -l app=user-service
```

---

## Configuration
### Environment Variables
| Variable                | Description                          | Default       | Required |
|-------------------------|--------------------------------------|---------------|----------|
| `SPRING_PROFILES_ACTIVE`| Active Spring profile                | `dev`         | No       |
| `DB_URL`                | PostgreSQL connection URL            | -             | Yes      |
| `DB_USERNAME`           | Database username                    | -             | Yes      |
| `REDIS_HOST`            | Redis server address                 | `localhost`   | No       |
| `KEYCLOAK_URL`          | Keycloak auth server URL             | -             | Yes      |

### Profiles
- **`dev`**: Embedded H2 DB, console logging
- **`staging`**: External DB, structured JSON logs
- **`prod`**: Full external dependencies, TLS enabled

---

## Health Checks & Monitoring
### Endpoints
| Endpoint               | Method | Description                     |
|------------------------|--------|---------------------------------|
| `/actuator/health`     | GET    | Service health status           |
| `/actuator/info`       | GET    | Build/version info              |
| `/actuator/prometheus` | GET    | Metrics for Prometheus scraping |

### Critical Metrics
- **`http.server.requests`**: Request latency/error rates
- **`jdbc.connections.active`**: DB connection pool usage
- **`cache.gets.miss`**: Redis cache miss ratio

### Alert Thresholds
- **CPU > 80%** for 5 minutes
- **HTTP 5xx errors > 1%** of total requests
- **DB connection pool > 90%** utilization

---

## Logging
### Log Levels
- **Production**: `INFO` (structured JSON)
- **Development**: `DEBUG` (human-readable)

### Log Locations
- **Docker**: `stdout`/`stderr`
- **Kubernetes**: `kubectl logs <pod-name>`
- **File System**: `/var/log/user-service/app.log` (if file appender enabled)

### Key Log Patterns
```log
# Authentication failure
"event":"AUTH_FAILURE","userId":"usr_123","ip":"192.168.1.10"

# DB query timeout
"event":"DB_TIMEOUT","query":"SELECT * FROM users WHERE id=?"
```

---

## Scaling
### Horizontal Scaling
- **Kubernetes**: Adjust replicas in `Deployment`:
  ```yaml
  spec:
    replicas: 3  # Scale as needed
  ```
- **Cloud**: Use auto-scaling groups (AWS ASG/GCP MIG)

### Vertical Scaling
- Increase CPU/memory limits in container specs:
  ```yaml
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  ```

---

## Troubleshooting
### Common Issues
| Symptom                          | Diagnosis Steps                                                                 | Resolution                                  |
|----------------------------------|---------------------------------------------------------------------------------|---------------------------------------------|
| **500 Errors on login**          | 1. Check Keycloak connectivity<br>2. Verify JWT signing key                     | Update Keycloak URL or refresh signing key  |
| **High DB latency**              | 1. Check `jdbc.connections.active`<br>2. Analyze slow queries                   | Optimize queries or scale DB                |
| **Cache misses > 30%**           | 1. Verify Redis connectivity<br>2. Check cache TTL settings                     | Increase cache size or adjust TTL           |
| **Pod crashes (OOMKilled)**      | 1. Check memory usage metrics<br>2. Review heap dump                             | Increase memory limit or fix memory leak    |

### Diagnostic Commands
```bash
# Check service logs
kubectl logs -l app=user-service --tail=100

# Test health endpoint
curl http://user-service:8080/actuator/health

# Describe pod events
kubectl describe pod <user-service-pod>
```

---

## Rollback Procedure
### Kubernetes Rollback
```bash
# Rollback to previous revision
kubectl rollout undo deployment/user-service

# Verify rollback
kubectl rollout status deployment/user-service
```

### Manual Rollback (Docker)
1. Pull previous image version:
   ```bash
   docker pull user-service:1.1.0
   ```
2. Stop current container and start previous version with same env vars

### Verification Steps
1. Confirm version via `/actuator/info`
2. Validate critical endpoints (login, profile fetch)
3. Monitor error rates for 15 minutes

---

## Contact Information
| Role                   | Contact                     | Escalation Path          |
|------------------------|-----------------------------|--------------------------|
| **Primary On-Call**    | Jane Doe (jane@company.com) | Slack: `@platform-team`  |
| **Backup On-Call**     | John Smith (john@company.com)| PagerDuty: `user-svc`   |
| **Dev Team Lead**      | Alex Chen (alex@company.com)| Email only               |

**Incident Response**:  
- **P0 (Service Down)**: Page on-call immediately  
- **P1 (Degraded)**: Slack alert + email within 1 hour  
- **P2 (Minor Issue)**: Jira ticket within 24 hours  

> **Note**: Always update this runbook after major changes!