# Spring Boot + Kubernetes with Docker Desktop on Linux

## Prerequisites

1. **Docker Desktop for Linux** installed and running
2. **Kubernetes enabled** in Docker Desktop settings
3. **Java 17+** and **Maven** or **Gradle**
4. **kubectl** command-line tool

## Step 1: Create a Spring Boot Application

### Using Spring Boot CLI or Spring Initializr

```bash
# Using Spring Initializr (web)
# Go to https://start.spring.io/
# Or use curl:
curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator \
  -d javaVersion=17 \
  -d bootVersion=3.2.0 \
  -d name=demo-app \
  -d artifactId=demo-app \
  -d packageName=com.example.demo \
  -o demo-app.zip

unzip demo-app.zip
cd demo-app
```

### Sample Application Code

**src/main/java/com/example/demo/DemoApplication.java**
```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

@RestController
class HelloController {
    
    @GetMapping("/")
    public String hello() {
        return "Hello from Spring Boot on Kubernetes!";
    }
    
    @GetMapping("/health")
    public String health() {
        return "Application is healthy!";
    }
}
```

**src/main/resources/application.yml**
```yaml
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always

logging:
  level:
    com.example.demo: INFO
```

## Step 2: Create Dockerfile

**Dockerfile**
```dockerfile
# Multi-stage build
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

# Build the application
RUN ./mvnw clean package -DskipTests

# Runtime stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Copy the built JAR
COPY --from=builder /app/target/*.jar app.jar

# Change ownership
RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Step 3: Build and Test Docker Image

```bash
# Build the Docker image
docker build -t demo-app:1.0.0 .

# Test locally
docker run -p 8080:8080 demo-app:1.0.0

# Test the endpoints
curl http://localhost:8080
curl http://localhost:8080/health
curl http://localhost:8080/actuator/health
```

## Step 4: Kubernetes Deployment Files

### Namespace
**k8s/namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
  labels:
    name: demo-app
```

### ConfigMap
**k8s/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app-config
  namespace: demo-app
data:
  application.yml: |
    server:
      port: 8080
    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus
      endpoint:
        health:
          show-details: always
    logging:
      level:
        com.example.demo: INFO
        org.springframework: INFO
```

### Deployment
**k8s/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: demo-app
  labels:
    app: demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: demo-app:1.0.0
        imagePullPolicy: Never  # Use local image
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
      volumes:
      - name: config-volume
        configMap:
          name: demo-app-config
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
  namespace: demo-app
  labels:
    app: demo-app
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: demo-app
```

### Ingress (Optional)
**k8s/ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-app-ingress
  namespace: demo-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: demo-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-app-service
            port:
              number: 80
```

### Service (LoadBalancer for easy access)
**k8s/service-lb.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app-loadbalancer
  namespace: demo-app
  labels:
    app: demo-app
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: demo-app
```

## Step 5: Deploy to Kubernetes

```bash
# Verify Kubernetes is running
kubectl cluster-info

# Apply all Kubernetes manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service-lb.yaml

# Check deployment status
kubectl get all -n demo-app

# Watch pods starting up
kubectl get pods -n demo-app -w

# Check logs
kubectl logs -n demo-app -l app=demo-app

# Get service details
kubectl get svc -n demo-app
```

## Step 6: Access Your Application

```bash
# Method 1: Using LoadBalancer service
kubectl get svc demo-app-loadbalancer -n demo-app

# Method 2: Port forwarding
kubectl port-forward -n demo-app svc/demo-app-service 8080:80

# Method 3: Using kubectl proxy
kubectl proxy &
# Then access: http://localhost:8001/api/v1/namespaces/demo-app/services/demo-app-service:80/proxy/

# Test the application
curl http://localhost:8080
curl http://localhost:8080/actuator/health
```

## Step 7: Useful Commands

### Scaling
```bash
# Scale deployment
kubectl scale deployment demo-app --replicas=5 -n demo-app

# Auto-scaling (HPA)
kubectl autoscale deployment demo-app --cpu-percent=70 --min=2 --max=10 -n demo-app
```

### Monitoring
```bash
# Watch pods
kubectl get pods -n demo-app -w

# Describe deployment
kubectl describe deployment demo-app -n demo-app

# View logs
kubectl logs -f -n demo-app -l app=demo-app

# Execute commands in pod
kubectl exec -it -n demo-app <pod-name> -- /bin/sh
```

### Updates
```bash
# Update image
kubectl set image deployment/demo-app demo-app=demo-app:2.0.0 -n demo-app

# Rollback
kubectl rollout undo deployment/demo-app -n demo-app

# Check rollout status
kubectl rollout status deployment/demo-app -n demo-app
```

## Step 8: Production Considerations

### Security
```yaml
# Add to deployment.yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  fsGroup: 1001
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
```

### Resource Management
```yaml
# Resource quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-app-quota
  namespace: demo-app
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    persistentvolumeclaims: "2"
```

### Monitoring with Prometheus
```yaml
# Add to service metadata
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: "/actuator/prometheus"
  prometheus.io/port: "8080"
```

## Troubleshooting

### Common Issues

1. **Image not found**: Ensure `imagePullPolicy: Never` for local images
2. **Port conflicts**: Check if port 8080 is already in use
3. **Resource constraints**: Adjust CPU/memory limits
4. **Health check failures**: Increase `initialDelaySeconds`

### Debug Commands
```bash
# Check events
kubectl get events -n demo-app --sort-by='.lastTimestamp'

# Describe problematic pods
kubectl describe pod <pod-name> -n demo-app

# Check resource usage
kubectl top pods -n demo-app
kubectl top nodes
```

## Cleanup

```bash
# Delete all resources
kubectl delete namespace demo-app

# Or delete individual resources
kubectl delete -f k8s/
```

This setup provides a production-ready Spring Boot application running on Kubernetes with proper health checks, resource management, and scaling capabilities.