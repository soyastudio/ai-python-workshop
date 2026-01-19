# DOCT-c360-api Runbook

## 1. Overview
**Service Name:** DOCT-c360-api  
**Description:** Customer 360 Profile API supporting low-latency, real-time business use cases  
**Version:** 1.0.2-SNAPSHOT  
**Technology Stack:** Spring Boot 3.5.3, Java 17, MongoDB, Redis, Azure Key Vault  

## 2. Prerequisites

### 2.1 Software Requirements
- **Java:** JDK 17 or higher
- **Maven:** 3.6+ (for local builds)
- **Docker:** (optional, for containerized deployment)
- **Azure CLI:** (for accessing Azure resources)
- **MongoDB:** 4.4+ (external dependency)
- **Redis:** 6.0+ with cluster support (external dependency)

### 2.2 Environment Variables Required
```bash
export MONGODB_URL="mongodb://your-mongo-connection-string"
export C360_REDIS_HOST="redis-host"
export C360_REDIS_PORT="6379"
export C360_REDIS_ACCESS_KEY="redis-password"
export C360_REDIS_TTL="6000000"
export C360_REDIS_SWITCH="true"
export C360_REDIS_NODES="node1:port,node2:port"
```

## 3. Build Process

### 3.1 Local Development Build
```bash
# Clone repository
git clone <repository-url>
cd doct-c360-api

# Build with Maven
mvn clean install -DskipTests

# Build with tests
mvn clean install

# Run tests only
mvn test

# Generate JaCoCo coverage report
mvn jacoco:report
```

### 3.2 CI/CD Pipeline Build
The service uses Maven with the following key plugins:
- **Spring Boot Maven Plugin:** For packaging executable JAR
- **JaCoCo Maven Plugin:** For code coverage reporting
- **Maven Surefire Plugin:** For unit tests

**Build Command:**
```bash
mvn clean package -DskipTests -Pprod
```

**SonarQube Integration:**
```bash
mvn sonar:sonar
```

## 4. Deployment

### 4.1 Configuration Management

#### 4.1.1 Azure Key Vault Integration
- Configuration properties are stored in Azure Key Vault
- Secrets are injected during CI/CD pipeline execution
- Supported properties (see `application.yaml`):
  - `mongodb-url`
  - `c360-redis-host`, `c360-redis-port`, `c360-redis-access-key`
  - `c360-redis-ttl`, `c360-redis-switch`, `c360-redis-nodes`

#### 4.1.2 Profile Configuration
- **Active Profile:** `prod` (set via `spring.profiles.active`)
- **Configuration File:** `application.yaml` (prod-specific configuration)

### 4.2 Deployment Methods

#### 4.2.1 Standalone JAR Deployment
```bash
# Build executable JAR
mvn clean package -DskipTests -Pprod

# Run application
java -jar target/doct-c360-api-1.0.2-SNAPSHOT.jar \
  --spring.config.location=classpath:application.yaml \
  --spring.profiles.active=prod
```

#### 4.2.2 Docker Container Deployment
```dockerfile
# Sample Dockerfile (not provided but recommended)
FROM openjdk:17-jdk-slim
COPY target/doct-c360-api-1.0.2-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

#### 4.2.3 Kubernetes Deployment (Recommended for Production)
```yaml
# Sample Kubernetes deployment manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: doct-c360-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: doct-c360-api
  template:
    metadata:
      labels:
        app: doct-c360-api
    spec:
      containers:
      - name: doct-c360-api
        image: doct-c360-api:1.0.2
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: MONGODB_URL
          valueFrom:
            secretKeyRef:
              name: c360-secrets
              key: mongodb-url
        # Additional environment variables from Azure Key Vault
```

## 5. Service Startup & Validation

### 5.1 Startup Sequence
1. **Check dependencies:** MongoDB and Redis connectivity
2. **Load configuration:** From Azure Key Vault and application.yaml
3. **Initialize components:**
   - MongoDB repositories
   - Redis cache configuration
   - WebFlux reactive web server
4. **Start application:** On port 8080 (configurable)

### 5.2 Health Check Endpoints
```bash
# Health check
curl http://localhost:8080/actuator/health

# Info endpoint
curl http://localhost:8080/actuator/info

# Metrics (Prometheus format)
curl http://localhost:8080/actuator/prometheus

# API Documentation (OpenAPI/Swagger)
curl http://localhost:8080/v3/api-docs
```

### 5.3 Service Validation
```bash
# Test API endpoints
# Get customer info (V3)
curl -X GET "http://localhost:8080/doct-c360-api/v3/customer_info/{uuid}" \
  -H "accept: application/hal+json"

# Get customer transaction aggregates
curl -X GET "http://localhost:8080/doct-c360-api/v1/customer_transaction_aggregates/{uuid}?startDate=2024-01-01&endDate=2024-12-31" \
  -H "accept: application/hal+json"

# Get catalog metadata
curl -X GET "http://localhost:8080/doct-c360-api/v3/catalog/all" \
  -H "accept: application/hal+json"
```

## 6. API Documentation

### 6.1 API Versions
- **V3 APIs:** Latest version with enhanced schemas
- **V2 APIs:** Previous version (maintains backward compatibility)
- **V1 APIs:** Legacy endpoints (deprecated but supported)

### 6.2 Key Endpoints

| Endpoint | Method | Description | Version |
|----------|--------|-------------|---------|
| `/v3/customer_info/{uuid}` | GET | Complete customer profile | V3 |
| `/v3/catalog/all` | GET | Catalog metadata | V3 |
| `/v2/customer_info/{uuid}` | GET | Customer profile (V2 schema) | V2 |
| `/v1/customer_transaction_aggregates/{uuid}` | GET | Transaction aggregates with date filters | V1 |
| `/v1/discovery/CDPSegments` | GET | Customer Data Platform segments | V1 |
| `/v1/bnc/{uuid}` | GET | BNC customer transaction data | V1 |

### 6.3 Response Formats
- **Content-Type:** `application/hal+json`
- **Schema:** Defined in OpenAPI specification (`api-docs.json`)
- **Error Responses:** Standard HTTP status codes

## 7. Monitoring & Observability

### 7.1 Logging Configuration
```yaml
# Log levels (from application.yaml)
logging.level:
  ROOT: INFO
  io.github.jhipster: INFO
  com.albertsons.c360.api: INFO
```

### 7.2 Metrics Collection
- **Framework:** Micrometer with Prometheus registry
- **Endpoints:** `/actuator/prometheus`
- **Integration:** Compatible with Prometheus/Grafana

### 7.3 Distributed Tracing
- **Framework:** Spring Cloud Sleuth
- **Sampling Rate:** 100% (configurable)
- **Trace Export:** Integrated with monitoring system

## 8. Performance & Scaling

### 8.1 Caching Strategy
- **Cache Provider:** Redis Cluster
- **TTL Configuration:** 6,000,000 ms (~100 minutes)
- **Cache Null Values:** Disabled
- **Key Prefix:** Enabled
- **SSL:** Enabled for secure connections

### 8.2 Database Optimization
- **MongoDB Read Preference:** Primary
- **Connection Pool:** Configured via MongoDB URI
- **Indexing:** Ensure proper indexes on frequently queried fields

### 8.3 Scaling Considerations
- **Horizontal Scaling:** Stateless service, can scale horizontally
- **Session Management:** No session state stored locally
- **Cache Sharing:** Redis cluster enables cache sharing across instances

## 9. Troubleshooting

### 9.1 Common Issues

#### Issue 1: MongoDB Connection Failure
**Symptoms:**
- Application fails to start
- `MongoTimeoutException` in logs

**Resolution:**
1. Verify MongoDB URL in Azure Key Vault
2. Check network connectivity to MongoDB cluster
3. Validate authentication credentials

#### Issue 2: Redis Cache Connection Failure
**Symptoms:**
- Cache operations failing
- `RedisConnectionException` in logs

**Resolution:**
1. Verify Redis cluster configuration
2. Check SSL certificate validity
3. Validate Redis access key in Key Vault

#### Issue 3: API Response Time Degradation
**Symptoms:**
- Increased latency in API responses
- Timeout errors

**Resolution:**
1. Check Redis cache hit/miss ratio
2. Monitor MongoDB query performance
3. Review application logs for slow operations

### 9.2 Diagnostic Commands
```bash
# Check application logs
kubectl logs deployment/doct-c360-api -f

# Check health status
curl http://localhost:8080/actuator/health

# Check thread dump
curl http://localhost:8080/actuator/threaddump

# Check environment
curl http://localhost:8080/actuator/env
```

## 10. Disaster Recovery

### 10.1 Backup Procedures
- **Configuration:** Backed up in Azure Key Vault
- **Application Code:** Version controlled in Git repository
- **Database:** MongoDB backup procedures should be established separately

### 10.2 Recovery Steps
1. **Service Failure:**
   - Restart affected pod/container
   - Check dependency services (MongoDB, Redis)
   - Verify configuration from Key Vault

2. **Data Corruption:**
   - Restore from MongoDB backups
   - Clear Redis cache if necessary
   - Restart application

## 11. Security

### 11.1 Authentication & Authorization
- **External Dependencies:** Assumes API gateway handles authentication
- **Service-to-Service:** Secure communication via SSL/TLS

### 11.2 Secret Management
- **Primary Store:** Azure Key Vault
- **Rotation Policy:** Regular rotation of Redis access keys
- **Access Control:** Role-based access to Key Vault secrets

### 11.3 Network Security
- **Internal Service:** Should not be exposed directly to internet
- **API Gateway:** All traffic should route through API gateway
- **SSL/TLS:** Enabled for Redis connections

## 12. Maintenance & Updates

### 12.1 Version Upgrades
1. **Backward Compatibility:** Ensure API versioning maintains compatibility
2. **Database Migration:** Review schema changes between versions
3. **Rolling Deployment:** Deploy new version with zero downtime

### 12.2 Scheduled Maintenance
- **Redis Cache Clearance:** May be required after data model changes
- **MongoDB Index Optimization:** Regular index maintenance
- **Log Rotation:** Configured at infrastructure level

---

## Appendix

### A. Dependencies Matrix
| Component | Version | Purpose |
|-----------|---------|---------|
| Spring Boot | 3.5.3 | Application framework |
| Spring Data MongoDB | 3.5.3 | MongoDB integration |
| Spring Data Redis | 3.5.3 | Redis cache integration |
| Spring WebFlux | 3.5.3 | Reactive web layer |
| OpenAPI | 2.8.13 | API documentation |
| Lettuce | Latest | Redis client |
| Azure Key Vault | 2.2.1 | Secret management |

### B. Performance Benchmarks
- **Expected Latency:** < 100ms for cache hits
- **Throughput:** 1000+ requests/second per instance
- **Cache Hit Ratio:** Target > 90%

### C. Contact Information
- **Support Team:** C360 Support Team
- **Email:** Customerc360@albertsons.com
- **URL:** https://albertsons.com

---

**Last Updated:** December 1, 2025  
**Document Version:** 1.0