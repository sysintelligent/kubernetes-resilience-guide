# The 12 Factors of Kubernetes Resilience

As Kubernetes adoption surges in cloud environments, ensuring applications are resilient to downtime—whether from node failures, upgrades, or network issues—is critical for meeting Service Level Agreements (SLAs) with customers. A single minute of downtime can cost enterprises $100,000+ in lost revenue and eroded customer trust [Gartner, 2014], while 99.9% availability allows only 8.76 hours of downtime per year compared to 4.38 minutes for 99.99% availability [Uptime Institute, 2021].

This guide presents **The 12 Factors of Kubernetes Resilience**—a comprehensive methodology for building unstoppable applications in Kubernetes. Inspired by the 12-Factor App methodology, these factors provide a systematic approach to achieving 99.9%+ availability through proper pod distribution, health checks, autoscaling, and application-level patterns like circuit breakers and retries.

Each factor builds upon the previous ones, creating a layered approach to resilience that addresses both infrastructure and application concerns. Updated for Kubernetes 1.28+, this guide shows how to achieve production-ready resilience through intentional design at both the application and infrastructure layers.

## The 12 Factors Overview

- [Factor I: Replicated Workloads](#factor-i-replicated-workloads) - Eliminate single points of failure
- [Factor II: Health Monitoring](#factor-ii-health-monitoring) - Detect and recover from failures  
- [Factor III: Resource Management](#factor-iii-resource-management) - Prevent resource exhaustion and enable optimization
- [Factor IV: Graceful Lifecycle](#factor-iv-graceful-lifecycle) - Manage pod startup and shutdown
- [Factor V: Topology Distribution](#factor-v-topology-distribution) - Survive infrastructure failures
- [Factor VI: Auto-Scaling](#factor-vi-auto-scaling) - Handle traffic spikes and failures
- [Factor VII: Disruption Protection](#factor-vii-disruption-protection) - Protect during planned maintenance
- [Factor VIII: Configuration Resilience](#factor-viii-configuration-resilience) - Manage configuration changes safely
- [Factor IX: Storage Resilience](#factor-ix-storage-resilience) - Protect data and state
- [Factor X: Network Resilience](#factor-x-network-resilience) - Control traffic and communication
- [Factor XI: Security Resilience](#factor-xi-security-resilience) - Prevent security-related failures
- [Factor XII: Application Resilience](#factor-xii-application-resilience) - Handle application-level failures

## Quick Start: Production-Ready Resilient Configuration

For teams starting their resilience journey, here's a comprehensive, production-ready configuration that implements multiple factors:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resilient-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resilient-app
  template:
    metadata:
      labels:
        app: resilient-app
    spec:
      # Enhanced pod distribution across nodes and zones
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          # Node-level distribution (critical for single-zone clusters)
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: resilient-app
              topologyKey: kubernetes.io/hostname
          # Zone-level distribution for multi-zone clusters
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: resilient-app
              topologyKey: topology.kubernetes.io/zone
      # Topology spread constraints for even better distribution
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: resilient-app
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: resilient-app:latest
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        # Startup probe for slow-starting applications
        startupProbe:
          httpGet:
            path: /health
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        # Enhanced health checks with timeout values
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        # Graceful shutdown lifecycle management
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                echo "Graceful shutdown initiated"
                # Give time for load balancer to remove from rotation
                sleep 5
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: resilient-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: resilient-app
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: resilient-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: resilient-app
  minReplicas: 3  # Increased to maintain HA during scaling
  maxReplicas: 15  # Increased max capacity
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Add memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Controlled scaling behavior to prevent thrashing
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: resilient-app-service
spec:
  selector:
    app: resilient-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### Why This Configuration is Production-Ready

This configuration implements multiple layers of resilience that work together to ensure high availability:

#### **Multi-Layer Pod Distribution**
```yaml
# Node-level distribution
podAntiAffinity:
  topologyKey: kubernetes.io/hostname  # Spreads across different nodes

# Zone-level distribution  
podAntiAffinity:
  topologyKey: topology.kubernetes.io/zone  # Spreads across availability zones

# Even distribution enforcement
topologySpreadConstraints:
  maxSkew: 1  # Ensures balanced pod distribution
```
**Impact**: Prevents cascading failures when nodes or entire zones go down. If one availability zone fails, your application remains available from other zones.

#### **Comprehensive Health Monitoring**
```yaml
startupProbe:     # Handles slow application startup (up to 5 minutes)
livenessProbe:    # Detects and restarts unhealthy containers
readinessProbe:   # Controls traffic routing to healthy pods only
```
**Impact**: 
- **Startup probe** prevents premature restarts during slow application initialization
- **Separate endpoints** (`/health` vs `/ready`) allow granular control over container lifecycle vs traffic routing
- **Timeout values** prevent hanging health checks that could mask real issues

#### **Graceful Shutdown Management**
```yaml
terminationGracePeriodSeconds: 60  # Extended time for cleanup
lifecycle:
  preStop:  # Coordinated shutdown sequence
    - sleep 5  # Allow load balancer to drain connections
```
**Impact**: Prevents dropped requests during deployments, scaling, or node maintenance by ensuring clean connection termination.

#### **Intelligent Auto-Scaling**
```yaml
minReplicas: 3      # Always maintain HA baseline
maxReplicas: 15     # Handle traffic spikes
metrics:
  - cpu: 70%        # Scale on CPU pressure
  - memory: 80%     # Scale on memory pressure
behavior:           # Prevent scaling thrashing
  scaleDown: 10%/min  # Gradual scale-down
  scaleUp: 50%/min    # Rapid scale-up for traffic spikes
```
**Impact**: 
- **Dual metrics** ensure scaling responds to both CPU and memory pressure
- **Controlled scaling** prevents resource waste and instability
- **Higher minimum** (3 vs 2) maintains availability during scaling events

#### **Disruption Protection**
```yaml
PodDisruptionBudget:
  minAvailable: 2  # Always keep 2 pods running during maintenance
```
**Impact**: Ensures service availability during cluster maintenance, node upgrades, or voluntary disruptions.

#### **Cost-Efficiency Features**
- **Resource limits** prevent runaway containers from consuming excessive resources
- **Smart scaling** reduces over-provisioning while maintaining performance
- **Multi-zone distribution** balances availability with cross-zone data transfer costs

This configuration achieves **99.9%+ availability** by eliminating single points of failure and automatically recovering from common failure scenarios including node failures, application crashes, traffic spikes, and planned maintenance.

---

# Factor I: Replicated Workloads

**Purpose**: Eliminate single points of failure through multiple pod replicas across failure domains

**Priority**: Critical Foundation

**Why Essential**: Without replicas, any single pod failure causes complete service outage. This is the **most fundamental** resilience feature - the foundation upon which all other factors build.

Running multiple pod replicas ensures redundancy, allowing the application to remain available if some pods fail. This factor addresses the core principle of eliminating single points of failure.

**Use Case**: Essential for all production workloads; stateless apps (e.g., web servers) use `Deployments`; stateful apps (e.g., databases) use `StatefulSets`

**Important Exception**: **Kubernetes Operators and Controllers** - This factor does not apply to Kubernetes operators and custom controllers that follow the reconciliation pattern. These components are designed to run as single instances and rely on the controller pattern for resilience rather than traditional replication. The Kubernetes control plane itself handles operator failures through the reconciliation loop, where the desired state is continuously reconciled regardless of individual controller pod failures.

**Configuration**:
- Set `replicas: N` (minimum 3 for production) to maintain multiple pods
- For stateful apps, use `StatefulSet` with stable pod identities and persistent storage
- **Best Practice**: Start with at least 3 replicas (not 2) to handle rolling updates without downtime
- **Impact**: **Eliminates single points of failure** - if one pod crashes, others continue serving traffic

**Example YAML**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # Critical: Minimum for zero-downtime updates
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: my-web-app:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        # Basic health checks for Factor II integration
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
```

---

# Factor II: Health Monitoring

**Purpose**: Detect and recover from failures through comprehensive health checks and probes

**Priority**: Critical Foundation

**Why Essential**: Health checks are the **second most critical** feature for resilience. Without proper health checks, Kubernetes cannot detect failed pods or route traffic away from unhealthy instances, leading to user-facing errors.

Probes ensure Kubernetes only routes traffic to healthy pods and restarts unhealthy ones. This factor builds upon Factor I (Replicated Workloads) by ensuring that only healthy replicas receive traffic.

**Use Case**: Essential for detecting app crashes, slow startups, or degraded performance

**Configuration**:
- **Liveness**: Restarts containers when they become unresponsive
- **Readiness**: Controls traffic routing to healthy pods
- **Startup**: Handles slow-starting containers (Kubernetes 1.16+)
- **Best Practice**: Expose reliable `/health` and `/ready` endpoints; tune timeouts for your application
- **Impact**: **Prevents routing traffic to failed pods** and enables automatic recovery

**Comprehensive Health Check Example**:

```yaml
containers:
- name: app
  image: my-app:latest
  
  # STARTUP PROBE - Handles slow-starting applications
  # Prevents premature restarts during application initialization
  startupProbe:
    httpGet:
      path: /startup          # Endpoint that indicates app is starting up
      port: 8080
    failureThreshold: 30      # Allow up to 30 failures (5 minutes total)
    periodSeconds: 10         # Check every 10 seconds
    timeoutSeconds: 5         # Fail if response takes longer than 5 seconds
  
  # LIVENESS PROBE - Detects and restarts unhealthy containers
  # Restarts the container if it becomes unresponsive or stuck
  livenessProbe:
    httpGet:
      path: /health           # Endpoint that checks basic application health
      port: 8080
    initialDelaySeconds: 15   # Wait 15 seconds before first check (after startup)
    periodSeconds: 10         # Check every 10 seconds
    timeoutSeconds: 5         # Fail if response takes longer than 5 seconds
    failureThreshold: 3       # Restart after 3 consecutive failures (30 seconds)
  
  # READINESS PROBE - Controls traffic routing to healthy pods
  # Only routes traffic to pods that are ready to serve requests
  readinessProbe:
    httpGet:
      path: /ready            # Endpoint that verifies all dependencies are available
      port: 8080
    initialDelaySeconds: 5    # Wait 5 seconds before first check (shorter than liveness)
    periodSeconds: 5          # Check every 5 seconds (more frequent for traffic control)
    timeoutSeconds: 3         # Fail if response takes longer than 3 seconds
    failureThreshold: 3       # Remove from traffic after 3 failures (15 seconds)
```

---

# Factor III: Resource Management

**Purpose**: Prevent resource exhaustion through proper resource requests, limits, and QoS classes

**Priority**: Critical Foundation

**Why Essential**: Without proper resource management, pods can consume excessive resources, causing node failures and cascading outages. This factor ensures stable resource allocation and prevents "noisy neighbor" problems that can cascade across the entire cluster.

Resource management works in conjunction with Factors I and II to ensure that replicated workloads have stable, predictable resource allocation and can be properly scheduled and monitored.

**Use Case**: All production workloads to prevent resource contention and ensure predictable performance

**Configuration**:
- Set appropriate CPU and memory requests and limits based on actual usage patterns
- Use QoS classes (Guaranteed, Burstable, BestEffort) strategically for workload prioritization
- Implement resource quotas and limit ranges at the namespace level
- Configure priority classes for critical workloads
- Monitor resource usage continuously and adjust limits based on observability data
- **Best Practice**: Start with conservative limits based on profiling, implement gradual right-sizing, and use VPA for optimization
- **Impact**: **Prevents resource exhaustion, ensures stable cluster performance, and enables predictable scaling behavior**

**1. CPU and Memory Requests and Limits**:

Start with basic resource management by setting CPU and memory requests and limits. This prevents resource exhaustion and ensures predictable performance.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3                    # Run 3 copies for high availability
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: my-app:latest
        ports:
        - containerPort: 8080    # Application listens on port 8080
        
        # Core resource management - REQUIRED for production
        resources:
          requests:              # Minimum resources needed to run
            cpu: "100m"          # 0.1 CPU cores (100 millicores) - aligned with other examples
            memory: "128Mi"      # 128 MiB RAM - minimum for stable operation
          limits:                # Maximum resources allowed
            cpu: "500m"          # 0.5 CPU cores - prevents CPU hogging
            memory: "512Mi"      # 512 MiB RAM - prevents OOM kills
        
        # Health checks - detect and restart unhealthy containers
        livenessProbe:           # Restart container if unhealthy
          httpGet:
            path: /health        # Health check endpoint
            port: 8080
          periodSeconds: 30      # Check every 30 seconds
          timeoutSeconds: 5      # Fail if response takes > 5 seconds
          failureThreshold: 3    # Restart after 3 consecutive failures
        
        readinessProbe:          # Control traffic routing
          httpGet:
            path: /ready         # Ready check endpoint
            port: 8080
          periodSeconds: 10      # Check every 10 seconds
          timeoutSeconds: 3      # Fail if response takes > 3 seconds
          failureThreshold: 3    # Remove from traffic after 3 failures
```

**2. QoS Classes and Their Impact**:

Use Quality of Service (QoS) classes to prioritize workloads under resource pressure. Choose the appropriate class based on your application's criticality.

```yaml
# Guaranteed QoS - Highest priority, last to be evicted
# Use for: Critical databases, monitoring systems, payment processors
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guaranteed-qos-app
spec:
  template:
    spec:
      containers:
      - name: critical-app
        image: critical-app:latest
        resources:
          requests:
            cpu: "500m"          # Minimum CPU needed
            memory: "512Mi"      # Minimum memory needed
          limits:
            cpu: "500m"          # MUST equal requests for Guaranteed QoS
            memory: "512Mi"      # MUST equal requests for Guaranteed QoS
---
# Burstable QoS - Medium priority, can burst above requests
# Use for: Web applications, APIs, most workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: burstable-qos-app
spec:
  template:
    spec:
      containers:
      - name: web-app
        image: web-app:latest
        resources:
          requests:
            cpu: "100m"          # Minimum CPU needed
            memory: "128Mi"      # Minimum memory needed
          limits:
            cpu: "500m"          # Can burst up to 5x requests
            memory: "256Mi"      # Can use up to 2x requests
---
# BestEffort QoS - Lowest priority, first to be evicted
# Use for: Batch jobs, non-critical workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: besteffort-qos-app
spec:
  template:
    spec:
      containers:
      - name: batch-job
        image: batch-job:latest
        # No resource specifications = BestEffort QoS
        # Gets whatever resources are available
```

**QoS Classes Priority and Eviction Order**:
- **Guaranteed** (requests = limits): Highest priority, last to be evicted
- **Burstable** (requests < limits): Medium priority, evicted before Guaranteed
- **BestEffort** (no requests/limits): Lowest priority, first to be evicted under pressure

**3. Resource Quotas and Limit Ranges**:

Implement namespace-level resource controls to prevent resource exhaustion and ensure fair allocation across teams or applications.

```yaml
# Resource Quota - Limits total resources in a namespace
# Use for: Multi-tenant environments, cost control, resource fairness
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:                                # Hard limits - cannot be exceeded
    requests.cpu: "8"                  # Total CPU requests across all pods
    requests.memory: 16Gi              # Total memory requests across all pods
    limits.cpu: "16"                   # Total CPU limits across all pods
    limits.memory: 32Gi                # Total memory limits across all pods
    requests.ephemeral-storage: 100Gi  # Total temporary storage
    limits.ephemeral-storage: 200Gi    # Total temporary storage limits
    persistentvolumeclaims: "10"       # Max number of PVCs
    services: "20"                     # Max number of services
    services.loadbalancers: "2"        # Max number of load balancers
    services.nodeports: "10"           # Max number of node ports
    count/pods: "50"                   # Max number of pods
    count/deployments.apps: "20"       # Max number of deployments
    count/statefulsets.apps: "5"       # Max number of statefulsets
    count/jobs.batch: "10"             # Max number of batch jobs
  scopes:                              # Which pods count toward quota
  - BestEffort                         # Include BestEffort pods
  - NotTerminating                     # Exclude pods that are terminating
---
# Limit Range - Sets defaults and limits for individual containers/pods
# Use for: Consistent resource allocation, preventing resource waste
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - default:                      # Default limits if not specified
      cpu: 500m                   # Default CPU limit per container
      memory: 512Mi               # Default memory limit per container
    defaultRequest:               # Default requests if not specified
      cpu: 100m                   # Default CPU request per container
      memory: 128Mi               # Default memory request per container
    type: Container               # Apply to individual containers
  - default:                      # Default limits per pod
      cpu: 1000m                  # Total CPU limit per pod
      memory: 1Gi                 # Total memory limit per pod
    defaultRequest:               # Default requests per pod
      cpu: 200m                   # Total CPU request per pod
      memory: 256Mi               # Total memory request per pod
    type: Pod                     # Apply to entire pods
  - type: PersistentVolumeClaim   # Storage limits
    max:
      storage: 100Gi              # Maximum PVC size
    min:
      storage: 1Gi                # Minimum PVC size
```

**4. Priority Classes for Critical Workloads**:

For critical workloads that must survive resource pressure, use priority classes to ensure they get scheduled first and are last to be evicted.

```yaml
# Priority Classes - Define scheduling priority for workloads
# Higher values = higher priority during scheduling and resource pressure

apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000000                    # Very high priority (1 million)
globalDefault: false              # Don't apply to all pods by default
description: "Critical workloads that should survive resource pressure"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000                     # High priority (100 thousand)
globalDefault: false              # Don't apply to all pods by default
description: "High priority workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000                       # Low priority (1 thousand)
globalDefault: false              # Don't apply to all pods by default
description: "Low priority workloads that can be preempted"
```

**Critical Workload with Priority**:

```yaml
# Critical Workload Example - Database with high priority
# This database will be scheduled first and evicted last
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-database
spec:
  replicas: 3                      # Run 3 database instances
  selector:
    matchLabels:
      app: critical-database
  template:
    metadata:
      labels:
        app: critical-database
    spec:
      priorityClassName: critical-priority  # Use critical priority class
      containers:
      - name: database
        image: postgres:14
        resources:
          requests:
            cpu: "1000m"           # 1 CPU core minimum
            memory: "2Gi"          # 2GB RAM minimum
          limits:
            cpu: "2000m"           # 2 CPU cores maximum
            memory: "4Gi"          # 4GB RAM maximum
```

**5. Resource Monitoring and Optimization**:

Monitor resource usage to identify optimization opportunities and prevent issues before they impact your application.

```yaml
# ServiceMonitor - Tells Prometheus to scrape metrics from your app
# Use for: Monitoring resource usage, performance metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app-metrics
  namespace: production
spec:
  selector:
    matchLabels:
      app: web-app                # Monitor pods with this label
  endpoints:
  - port: 8080                    # Port where metrics are exposed
    interval: 30s                 # Scrape every 30 seconds
    path: /metrics                # Metrics endpoint path
---
# PrometheusRule - Define alerts based on resource usage
# Use for: Proactive monitoring, early warning of resource issues
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: resource-alerts
  namespace: production
spec:
  groups:
  - name: resource-management
    rules:
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes{container="app"} / container_spec_memory_limit_bytes > 0.8
      for: 5m                      # Alert if condition persists for 5 minutes
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"  # Alert message
    
    - alert: HighCPUUsage
      expr: rate(container_cpu_usage_seconds_total{container="app"}[5m]) > 0.8
      for: 10m                     # Alert if condition persists for 10 minutes
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"     # Alert message
```

---

# Factor IV: Graceful Lifecycle

**Purpose**: Manage pod startup and shutdown gracefully to prevent dropped requests and ensure clean state transitions

**Priority**: High Priority Resilience

**Why Essential**: Proper lifecycle management is essential for **preventing dropped requests during deployments and ensuring reliable startup**. Without it, rolling updates can interrupt active connections, and slow-starting applications can cause cascading failures.

This factor builds upon the previous three factors by ensuring that the replicated workloads with proper health monitoring and resource management can transition between states without impacting user experience.

**Use Case**: Apps with active connections, in-flight requests, cleanup requirements, or slow startup times


```yaml
spec:
  terminationGracePeriodSeconds: 60  # Extended time for cleanup
  containers:
  - name: web-app
    image: my-app:latest
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    
    # STARTUP MANAGEMENT - Prevents premature restarts and ensures reliable initialization
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30    # Allow up to 5 minutes for startup
      periodSeconds: 10       # Check every 10 seconds
      timeoutSeconds: 5       # Fail if response takes > 5 seconds
    
    # HEALTH MONITORING (after startup) - Controls traffic routing and detects failures
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15  # Wait for startup to complete
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5   # Shorter delay for traffic routing
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    
    # SHUTDOWN MANAGEMENT - Prevents dropped requests during deployments
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Graceful shutdown initiated"
            # Stop accepting new connections
            curl -X POST http://localhost:8080/shutdown
            # Wait for in-flight requests to complete
            sleep 10
            echo "Shutdown complete"
```

**Key Timing Considerations:**
- **Startup Probe**: 0-5 minutes (prevents premature restarts)
- **Readiness Probe**: 5+ seconds (controls traffic routing)
- **Liveness Probe**: 15+ seconds (detects unhealthy containers)
- **Shutdown**: 60+ seconds (ensures clean termination)

---

# Factor V: Topology Distribution

**Purpose**: Survive infrastructure failures by distributing pods across nodes, zones, and regions

**Priority**: High Priority Resilience

**Why Essential**: Pod distribution is **critical for surviving infrastructure failures**. Without proper distribution, a single node or zone failure can take down your entire application, causing complete service outages.

This factor builds upon the previous factors by ensuring that replicated workloads are distributed across failure domains, not just running multiple copies on the same infrastructure.

**Use Case**: Essential for high-availability apps in multi-zone cloud clusters

**Configuration**:
- Use `topologySpreadConstraints` with `topologyKey: topology.kubernetes.io/zone`
  - Apply `podAntiAffinity` to spread pods across nodes
  - Consider `requiredDuringScheduling` vs `preferredDuringScheduling`
- **Best Practice**: Balance distribution with cloud costs (cross-zone traffic fees)
- **Impact**: **Ensures service survives node and zone failures**

**Key Decision Points**:
- **`DoNotSchedule` vs `ScheduleAnyway`**: 
  - `DoNotSchedule`: Strict enforcement - pods won't schedule if constraints can't be met (use for critical services)
  - `ScheduleAnyway`: Best effort - prefers constraint satisfaction but allows scheduling anyway (use for non-critical services)
- **`topologySpreadConstraints` vs `podAntiAffinity`**: 
  - `topologySpreadConstraints`: Better for even distribution across multiple domains
  - `podAntiAffinity`: Better for simple "avoid same node/zone" rules

**Comprehensive Topology Spreading and Node Affinity Example**:

Advanced pod distribution strategies for multi-zone, multi-region, and custom topology deployments.

- **Use Case**: High-availability applications, disaster recovery, workload optimization
- **Configuration**: Topology spread constraints, node affinity, pod anti-affinity, custom topology labels
- **Best Practice**: Use topology spreading for distribution, affinity for optimization, anti-affinity for isolation

```yaml
# Custom topology labels for rack-level distribution
apiVersion: v1
kind: Node
metadata:
  name: worker-1
  labels:
    rack: rack-a
    datacenter: dc-1
    tier: production
    topology.kubernetes.io/zone: us-east-1a
    topology.kubernetes.io/region: us-east-1
spec:
  taints: []
---
# Comprehensive deployment with topology spreading and affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-availability-app
  labels:
    app: high-availability-app
spec:
  replicas: 6  # Ensure enough replicas for meaningful distribution
  selector:
    matchLabels:
      app: high-availability-app
  template:
    metadata:
      labels:
        app: high-availability-app
    spec:
      # TOPOLOGY SPREAD CONSTRAINTS - Ensure even distribution across failure domains
      topologySpreadConstraints:
      # Zone-level distribution (critical for multi-zone clusters)
      - maxSkew: 1                                # Maximum difference in pod count between zones
        topologyKey: topology.kubernetes.io/zone  # Spread across availability zones
        whenUnsatisfiable: DoNotSchedule          # Fail scheduling if constraint can't be met
        labelSelector:
          matchLabels:
            app: high-availability-app
      # Node-level distribution (prevents all pods on same node)
      - maxSkew: 1                                # Maximum difference in pod count between nodes
        topologyKey: kubernetes.io/hostname       # Spread across different nodes
        whenUnsatisfiable: ScheduleAnyway         # Allow scheduling even if constraint can't be met
        labelSelector:
          matchLabels:
            app: high-availability-app
      # Region-level distribution (for multi-region deployments)
      - maxSkew: 1                                  # Maximum difference in pod count between regions
        topologyKey: topology.kubernetes.io/region  # Spread across different regions
        whenUnsatisfiable: DoNotSchedule            # Fail scheduling if constraint can't be met
        labelSelector:
          matchLabels:
            app: high-availability-app
      # Custom rack-level distribution (for on-premise clusters)
      - maxSkew: 1                         # Maximum difference in pod count between racks
        topologyKey: rack                  # Custom topology key for rack distribution
        whenUnsatisfiable: ScheduleAnyway  # Allow scheduling even if constraint can't be met
        labelSelector:
          matchLabels:
            app: high-availability-app
      # POD AFFINITY RULES - Control pod placement relative to other pods
      affinity:
        # Anti-affinity prevents pods from being scheduled together
        podAntiAffinity:
          # Preferred anti-affinity (soft constraint - will try to avoid but allow if needed)
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: high-availability-app
              topologyKey: kubernetes.io/hostname  # Avoid same node
          - weight: 50
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: high-availability-app
              topologyKey: topology.kubernetes.io/zone  # Avoid same zone (lower priority)
          # Required anti-affinity (hard constraint - must be satisfied)
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: high-availability-app
                tier: critical
            topologyKey: topology.kubernetes.io/zone  # Critical pods must be in different zones
        # Node affinity for hardware requirements and zone preferences
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values:
                - m5.large
                - m5.xlarge
                - m5.2xlarge
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-east-1a
                - us-east-1b
                - us-east-1c
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node.kubernetes.io/arch
                operator: In
                values:
                - amd64
          - weight: 50
            preference:
              matchExpressions:
              - key: rack
                operator: In
                values:
                - rack-a
                - rack-b
                - rack-c
      containers:
      - name: web-app
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

**GPU Workload Isolation with Taints and Tolerations**:

Specialized scheduling for GPU workloads with hardware isolation and resource optimization.

- **Use Case**: Machine learning workloads, GPU-intensive applications, hardware-specific deployments
- **Configuration**: Node taints, pod tolerations, GPU resource requests, specialized node affinity
- **Best Practice**: Use taints for hardware isolation, tolerations for workload placement

```yaml
# Node with GPU taint for workload isolation
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-1
  labels:
    hardware: gpu
    gpu-type: nvidia-tesla-v100
    topology.kubernetes.io/zone: us-east-1a
spec:
  taints:
  - key: hardware
    value: gpu
    effect: NoSchedule
---
# GPU workload with toleration and specialized scheduling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-inference
  template:
    metadata:
      labels:
        app: ml-inference
    spec:
      # Tolerations allow pods to be scheduled on tainted nodes
      tolerations:
      - key: hardware
        operator: Equal
        value: gpu
        effect: NoSchedule
      # Node affinity ensures placement on GPU nodes
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: hardware
                operator: In
                values:
                - gpu
              - key: gpu-type
                operator: In
                values:
                - nvidia-tesla-v100
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-east-1a
                - us-east-1b
                - us-east-1c
      containers:
      - name: inference
        image: ml-inference:latest
        resources:
          requests:
            nvidia.com/gpu: 1
            cpu: 500m
            memory: 2Gi
          limits:
            nvidia.com/gpu: 1
            cpu: 2000m
            memory: 8Gi
```

---

# Factor VI: Auto-Scaling

**Purpose**: Handle traffic spikes and failures through horizontal and vertical pod autoscaling

**Priority**: High Priority Resilience

**Why Essential**: HPA is **essential for handling traffic spikes and replacing failed pods**. Without autoscaling, traffic surges can overwhelm fixed-capacity deployments, causing cascading failures and service degradation.

This factor builds upon the previous factors by ensuring that the distributed, replicated workloads can automatically scale to handle varying load and replace failed instances.

**Use Case**: Essential for dynamic workloads like e-commerce APIs, web applications, or any service with variable traffic

**Configuration**:
- Scale replicas based on CPU, memory, or custom metrics
- Set appropriate min/max replica bounds
- Configure scaling behavior to prevent thrashing
- **Best Practice**: Combine with Cluster Autoscaler; use multiple metrics for robust scaling decisions
- **Impact**: **Automatically handles traffic spikes and maintains capacity during pod failures**

**Advanced HPA Example**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3  # Higher minimum for resilience
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  # Custom metrics for application-specific scaling
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: web-app-ingress
      target:
        type: Value
        value: 1000
  # External metrics for business-driven scaling
  - type: External
    external:
      metric:
        name: queue_length
        selector:
          matchLabels:
            queue: order-processing
      target:
        type: AverageValue
        averageValue: 100
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Prevent rapid scale-down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60   # Allow rapid scale-up
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
```

---

# Factor VII: Disruption Protection

**Purpose**: Protect workloads during planned maintenance through Pod Disruption Budgets and scheduling policies

**Priority**: High Priority Resilience

**Why Essential**: PDBs are **critical for maintaining availability during planned maintenance**. Without PDBs, cluster upgrades or node maintenance can take down all pods simultaneously, causing complete service outages.

This factor builds upon the previous factors by ensuring that the distributed, auto-scaling workloads are protected during intentional operational activities like cluster maintenance and node upgrades.

**Use Case**: Essential for all production services where downtime impacts SLAs

**Configuration**:
- Set `minAvailable` (e.g., 2 pods or 80%) or `maxUnavailable` (e.g., 1 pod)
- Apply to workloads via label selectors
- **Best Practice**: Use `minAvailable: 2` for critical services, avoid `minAvailable: 100%` which blocks all maintenance
- **Impact**: **Prevents cluster maintenance from causing service outages**

**Basic PDB Example**:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

**Advanced PDB Examples**:

```yaml
# Percentage-based PDB for scalable workloads
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb-percentage
spec:
  minAvailable: 80%  # Always keep 80% of pods available
  selector:
    matchLabels:
      app: web-app
---
# Absolute PDB for critical services
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: database-pdb-absolute
spec:
  minAvailable: 2  # Always keep exactly 2 pods available
  selector:
    matchLabels:
      app: critical-database
---
# PDB for StatefulSet with complex selector
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: statefulset-pdb
spec:
  minAvailable: 1
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - database
      - cache
    - key: statefulset.kubernetes.io/pod-name
      operator: Exists
```

---

# Factor VIII: Configuration Resilience

**Purpose**: Manage configuration changes safely without requiring pod restarts or causing downtime

**Priority**: Medium Priority

**Why Essential**: Configuration management is **critical for avoiding downtime during updates**. Without externalized configuration and hot reloading, every config change requires pod restarts, causing service interruptions.

This factor builds upon the previous factors by ensuring that the resilient workloads can adapt to configuration changes without disrupting service availability.

**Use Case**: Essential for feature flags, database connections, API keys, and runtime settings

**Configuration**:
- Use `ConfigMaps` for non-sensitive data
- Use `Secrets` for sensitive information
- Implement configuration hot-reloading in applications
- **Best Practice**: Version your configs; implement validation; use config checksums
- **Impact**: **Eliminates downtime from configuration changes and enables faster deployments**

**Configuration Example**:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "db.example.com"
  feature.newUI: "true"
  logging.level: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database.password: <base64-encoded-password>
  api.key: <base64-encoded-key>
---
# Pod configuration
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

# Factor IX: Storage Resilience

**Purpose**: Protect data and state through persistent storage and backup strategies

**Priority**: Medium Priority

**Why Essential**: Data loss can be catastrophic for applications. Without proper storage resilience, infrastructure failures can result in permanent data loss, making all other resilience factors meaningless.

This factor builds upon the previous factors by ensuring that the resilient workloads have reliable, persistent storage that can survive infrastructure failures.

**Use Case**: Databases, file storage, session data

**Configuration**:
- Use cloud-managed databases when possible (AWS RDS, GCP Cloud SQL)
- For Kubernetes storage, use `StatefulSet` with `PersistentVolumeClaims`
- Choose appropriate `StorageClass` with replication
- **Best Practice**: Implement regular backups; test restore procedures
- **Impact**: **Protects against data loss and ensures business continuity**

**StatefulSet with Persistent Storage**:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  selector:
    matchLabels:
      app: database
  serviceName: database-headless
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

---

# Factor X: Network Resilience

**Purpose**: Control traffic and communication through service mesh, network policies, and advanced networking patterns

**Priority**: Medium Priority

**Why Essential**: Network failures and traffic issues can cascade across services. Without proper network resilience, a failure in one service can bring down entire application stacks.

This factor builds upon the previous factors by ensuring that the resilient workloads can communicate reliably and handle network-level failures gracefully.

**Use Case**: Microservices architectures requiring circuit breakers, retries, and load balancing

**Configuration**:
- Deploy Istio, Linkerd, or similar service mesh
- Configure retry policies, circuit breakers, and timeout policies
- Implement canary deployments and traffic splitting
- **Best Practice**: Start simple; add complexity as needed
- **Impact**: **Prevents network failures from cascading across services**

**Istio Traffic Policy Example**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: web-app-circuit-breaker
spec:
  host: web-app-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    circuitBreaker:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    retryPolicy:
      attempts: 3
      perTryTimeout: 2s
```

---

# Factor XI: Security Resilience

**Purpose**: Prevent security-related failures through Pod Security Standards, RBAC, and network policies

**Priority**: Medium Priority

**Why Essential**: Security is foundational to resilience - compromised pods can cause cascading failures and data breaches. Without proper security controls, the entire resilience architecture can be undermined.

This factor builds upon the previous factors by ensuring that the resilient workloads are protected from security threats that could compromise their availability and integrity.

**Use Case**: Production environments, compliance requirements, defense in depth

**Configuration**:
- Implement Pod Security Standards (Restricted, Baseline, Privileged)
- Configure security contexts for containers
- Use non-root users and read-only filesystems
- Configure network policies for traffic control
- Implement RBAC with least privilege principle

**Best Practice**: Start with Restricted PSS and relax as needed, use dedicated service accounts for each workload

**Impact**: **Prevents security breaches that could compromise resilience**

**Pod Security Standards and Security Contexts**:

Security is foundational to resilience - compromised pods can cause cascading failures and data breaches. Pod Security Standards provide a standardized way to enforce security policies across namespaces.

```yaml
# =============================================================================
# NAMESPACE CONFIGURATION WITH POD SECURITY STANDARDS
# =============================================================================
# This namespace enforces the most restrictive security profile (Restricted)
# Pod Security Standards (PSS) provide three levels: Privileged, Baseline, Restricted
# Restricted is the most secure and recommended for production workloads
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # ENFORCE: Pods that violate the policy will be rejected
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.24
    # AUDIT: Violations are logged but pods are still created
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.24
    # WARN: Violations trigger warnings but pods are still created
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.24
---
# =============================================================================
# SECURE POD CONFIGURATION WITH DEFENSE IN DEPTH
# =============================================================================
# This pod demonstrates comprehensive security hardening for production workloads
# Each security context setting reduces the attack surface and prevents privilege escalation
apiVersion: v1
kind: Pod
metadata:
  name: secure-web-app
  namespace: production
spec:
  # POD-LEVEL SECURITY CONTEXT
  # These settings apply to all containers in the pod
  securityContext:
    # CRITICAL: Prevent running as root user (UID 0)
    # This is the most important security setting
    runAsNonRoot: true
    # Run as specific non-root user (UID 1000)
    # Use dedicated UIDs for different applications
    runAsUser: 1000
    # Set group ID for the process
    runAsGroup: 3000
    # Set filesystem group ID for mounted volumes
    fsGroup: 2000
    # Apply seccomp security profile to restrict system calls
    # RuntimeDefault is the most secure option
    seccompProfile:
      type: RuntimeDefault
    # SECURITY: Remove all Linux capabilities
    # This prevents privilege escalation and reduces attack surface
    capabilities:
      drop:
      - ALL
  containers:
  - name: web-app
    image: secure-web-app:latest
    # CONTAINER-LEVEL SECURITY CONTEXT
    # These settings apply specifically to this container
    securityContext:
      # PREVENT PRIVILEGE ESCALATION
      # Even if the container has capabilities, it cannot gain more privileges
      allowPrivilegeEscalation: false
      # IMMUTABLE FILESYSTEM
      # Makes root filesystem read-only, preventing malicious file modifications
      # Applications can only write to mounted volumes
      readOnlyRootFilesystem: true
      # REDUNDANT NON-ROOT CHECK
      # Additional check to ensure container doesn't run as root
      runAsNonRoot: true
      # SPECIFIC USER ID
      # Run as the same user as pod-level context for consistency
      runAsUser: 1000
      # CONTAINER CAPABILITIES
      # Remove all Linux capabilities from this specific container
      # This is the most restrictive setting
      capabilities:
        drop:
        - ALL
    # VOLUME MOUNTS FOR WRITABLE DIRECTORIES
    # Since root filesystem is read-only, we need writable volumes for:
    # - /tmp: Temporary files and caching
    # - /var/log: Application logs
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: varlog
      mountPath: /var/log
  # VOLUME DEFINITIONS
  # These provide writable storage for the read-only container
  volumes:
  - name: tmp
    # Empty directory for temporary files
    emptyDir: {}
  - name: varlog
    # Empty directory for application logs
    emptyDir: {}
```

**Network Policies for Traffic Control**:

Network policies provide defense in depth by controlling traffic flow between pods, preventing lateral movement in case of compromise.

```yaml
# Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow web app to receive traffic from ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
---
# Allow web app to connect to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          name: cache
    ports:
    - protocol: TCP
      port: 6379
  - to: []  # Allow DNS resolution
    ports:
    - protocol: UDP
      port: 53
```

**RBAC and Service Account Management**:

Role-Based Access Control prevents unauthorized access and potential failures through proper access management using the principle of least privilege.

```yaml
# Service account for web application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
---
# Role for web application
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: web-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]
---
# Role binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-app-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: web-app-sa
  namespace: production
roleRef:
  kind: Role
  name: web-app-role
  apiGroup: rbac.authorization.k8s.io
---
# Cluster role for monitoring
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-role
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
---
# Cluster role binding for monitoring
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-rolebinding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-role
  apiGroup: rbac.authorization.k8s.io
```

**Deployment with Security Best Practices**:

Combining all security practices into a production-ready deployment configuration.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: web-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: web-app
        image: secure-web-app:latest
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: varlog
          mountPath: /var/log
        env:
        - name: PORT
          value: "8080"
        ports:
        - containerPort: 8080
          name: http
      volumes:
      - name: tmp
        emptyDir: {}
      - name: varlog
        emptyDir: {}
```

---

# Factor XII: Application Resilience

**Purpose**: Handle application-level failures through circuit breakers, retries, and resilience patterns that integrate with Kubernetes primitives

**Priority**: Advanced

**Why Essential**: Even with perfect infrastructure resilience, applications can fail due to logic errors, external dependency failures, or unexpected conditions. Application-level resilience patterns provide the final layer of protection and integrate seamlessly with Kubernetes health monitoring and traffic management.

This factor builds upon all previous factors by ensuring that the resilient infrastructure is complemented by resilient application code that can handle failures gracefully and work effectively with Kubernetes orchestration.

**Use Case**: All production applications, especially those with external dependencies, complex business logic, or high availability requirements

**Configuration**:
- Implement health check endpoints that integrate with Kubernetes probes
- Configure circuit breakers with appropriate failure thresholds
- Set up retry mechanisms with exponential backoff
- Implement graceful degradation for non-critical features
- Use structured logging for observability

**Best Practice**: Start with health checks and timeouts, then progressively add circuit breakers and advanced patterns based on your application's failure modes

**Impact**: **Prevents application-level failures from causing service outages** and enables graceful handling of dependency failures

**Resilience Patterns Overview**

| Priority | Pattern | Use Case | Failure Mode Addressed | Implementation Complexity |
|----------|---------|----------|----------------------|---------------------------|
| **Critical** | Health Check Endpoints | Kubernetes integration | Improper traffic routing | Low |
| **Critical** | Timeout Patterns | Slow/hanging operations | Resource exhaustion | Low |
| **Critical** | Graceful Degradation | Dependency failures | Complete service outages | Medium |
| **Essential** | Circuit Breaker | External dependencies | Cascading failures | Medium |
| **Essential** | Retry + Exponential Backoff | Transient errors | Temporary network issues | Low |
| **Essential** | Rate Limiting | High traffic/abuse | System overload | Medium |
| **Essential** | Bulkhead Pattern | Resource isolation | Noisy neighbor problems | High |
| **Production** | Idempotency | Duplicate operations | Data corruption | Medium |
| **Production** | Structured Logging | Debugging/monitoring | Poor observability | Low |
| **Production** | Async Processing | Workload distribution | Blocking operations | High |
| **Production** | Caching Patterns | Performance/availability | Backend overload | Medium |

**1. Health Check Endpoints**
**Priority: Critical** | **Why Essential**: Integrates with Kubernetes probes for proper traffic routing and pod lifecycle management.

Without proper health endpoints, Kubernetes cannot determine pod health, leading to traffic being routed to failed instances.

**Implementation**: Create separate endpoints for liveness (`/health`), readiness (`/ready`), and startup (`/startup`) probes. The liveness probe should check basic application health, readiness should verify all dependencies are available, and startup should handle slow-starting applications.

**Example Implementation**:

```yaml
# Kubernetes probe configuration
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 30
```

```python
# Health check endpoints implementation
from flask import Flask, jsonify
import redis
import requests

app = Flask(__name__)

@app.route('/health')
def health_check():
    """Liveness probe - basic application health"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat()
    })

@app.route('/ready')
def readiness_check():
    """Readiness probe - check all dependencies"""
    checks = {
        'database': check_database_connection(),
        'redis': check_redis_connection(),
        'external_api': check_external_api()
    }
    
    is_ready = all(checks.values())
    status_code = 200 if is_ready else 503
    
    return jsonify({
        'status': 'ready' if is_ready else 'not_ready',
        'checks': checks
    }), status_code

@app.route('/startup')
def startup_check():
    """Startup probe - handle slow initialization"""
    if app.config.get('initialization_complete', False):
        return jsonify({'status': 'started'})
    else:
        return jsonify({'status': 'starting'}), 503
```

**2. Timeout Patterns**
**Priority: Critical** | **Why Essential**: Prevents operations from hanging indefinitely during failures, avoiding resource exhaustion.

Without timeouts, a single slow dependency can cascade into system-wide failures as resources are exhausted.

**Implementation**: Implement timeout patterns at multiple levels - HTTP client timeouts, database query timeouts, and application-level operation timeouts. Use context managers and decorators to ensure consistent timeout handling across all external calls.

**Example Implementation**:

```python
import asyncio
import aiohttp
from contextlib import asynccontextmanager
from functools import wraps
import time

# HTTP client with timeouts
async def http_client_with_timeout():
    timeout = aiohttp.ClientTimeout(total=30, connect=10)
    async with aiohttp.ClientSession(timeout=timeout) as session:
        async with session.get('https://api.example.com/data') as response:
            return await response.json()

# Database timeout decorator
def with_timeout(timeout_seconds=30):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            try:
                return await asyncio.wait_for(func(*args, **kwargs), timeout=timeout_seconds)
            except asyncio.TimeoutError:
                raise TimeoutError(f"Operation {func.__name__} timed out after {timeout_seconds}s")
        return wrapper
    return decorator

@with_timeout(10)
async def database_query():
    # Database operation with timeout
    pass
```

**3. Graceful Degradation**
**Priority: Critical** | **Why Essential**: Maintains user experience when dependencies fail, preventing complete service outages.

Systems should provide reduced functionality rather than complete failure when non-critical dependencies are unavailable.

**Implementation**: Implement feature toggles and service level management that automatically disable non-critical features when dependencies fail. Provide fallback data and cached responses to maintain basic functionality during outages.

**Example Implementation**:

```python
from enum import Enum
from functools import wraps
import redis

class ServiceLevel(Enum):
    FULL = "full"
    DEGRADED = "degraded"
    MINIMAL = "minimal"

class GracefulDegradation:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.service_level = ServiceLevel.FULL
    
    def check_service_health(self):
        """Check all critical dependencies"""
        checks = {
            'database': self._check_database(),
            'cache': self._check_cache(),
            'external_api': self._check_external_api()
        }
        
        if all(checks.values()):
            self.service_level = ServiceLevel.FULL
        elif checks['database'] and checks['cache']:
            self.service_level = ServiceLevel.DEGRADED
        else:
            self.service_level = ServiceLevel.MINIMAL
        
        return self.service_level
    
    def with_fallback(self, fallback_func):
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    return fallback_func(*args, **kwargs)
            return wrapper
        return decorator

# Usage example
@graceful_degradation.with_fallback(lambda: {"data": "cached_data"})
def get_user_data(user_id):
    # Try to get fresh data
    return external_api.get_user(user_id)
```

**4. Retry Mechanisms with Exponential Backoff**
**Priority: Essential** | **Why Essential**: Handles transient failures in distributed systems, improving success rates for temporary network issues.

Most failures in cloud environments are transient. Proper retry logic dramatically improves reliability without masking real issues.

**Implementation**: Use exponential backoff with jitter to prevent thundering herd problems. Distinguish between transient and permanent errors, and implement smart retry logic that adapts based on error types and HTTP status codes.

**Example Implementation**:

```python
import random
import time
from functools import wraps
from typing import Callable, Any

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True
):
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    
                    if attempt == max_retries:
                        raise last_exception
                    
                    # Calculate delay with exponential backoff
                    delay = min(base_delay * (exponential_base ** attempt), max_delay)
                    
                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay = delay * (0.5 + random.random() * 0.5)
                    
                    time.sleep(delay)
            
            raise last_exception
        return wrapper
    return decorator

# Usage example
@retry_with_backoff(max_retries=3, base_delay=1.0)
def call_external_api():
    response = requests.get('https://api.example.com/data')
    response.raise_for_status()
    return response.json()
```

**5. Circuit Breaker Pattern**
**Priority: Essential** | **Why Essential**: Prevents cascading failures by isolating failing dependencies and enabling fast failure detection.

**Implementation**: Implement circuit breakers with three states (Closed, Open, Half-Open) that automatically isolate failing services and provide fallback responses. Configure appropriate failure thresholds and recovery timeouts based on service characteristics.

**Example Implementation**:

```python
from enum import Enum
import time
from typing import Callable, Any
from functools import wraps

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        expected_exception: type = Exception
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
    
    def call(self, func: Callable, *args, **kwargs) -> Any:
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except self.expected_exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Usage example
circuit_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=60)

def call_external_service():
    return circuit_breaker.call(
        lambda: requests.get('https://api.example.com/data').json()
    )
```

**6. Rate Limiting**
**Priority: Essential** | **Why Essential**: Protects services from overload and prevents cascading failures under high traffic or abuse.

Rate limiting is critical for preventing system overload and ensuring fair resource usage across clients.

**Implementation**: Implement token bucket or sliding window rate limiters with adaptive limits based on system health. Use distributed rate limiting with Redis for multi-instance deployments and priority-based rate limiting for different client types.

**Example Implementation**:

```python
import time
import redis
from typing import Optional

class RateLimiter:
    def __init__(self, redis_client: redis.Redis, max_requests: int = 100, window_seconds: int = 60):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_seconds = window_seconds
    
    def is_allowed(self, key: str) -> bool:
        """Check if request is allowed based on rate limit"""
        current_time = int(time.time())
        window_start = current_time - self.window_seconds
        
        # Remove old entries
        self.redis.zremrangebyscore(key, 0, window_start)
        
        # Count current requests
        current_requests = self.redis.zcard(key)
        
        if current_requests < self.max_requests:
            # Add current request
            self.redis.zadd(key, {str(current_time): current_time})
            self.redis.expire(key, self.window_seconds)
            return True
        
        return False

# Usage example
rate_limiter = RateLimiter(redis_client, max_requests=100, window_seconds=60)

def rate_limited_endpoint(user_id: str):
    if not rate_limiter.is_allowed(f"rate_limit:{user_id}"):
        raise Exception("Rate limit exceeded")
    
    # Process request
    return {"status": "success"}
```

**7. Bulkhead Pattern (Resource Isolation)**
**Priority: Essential** | **Why Essential**: Prevents failures in one component from affecting others by isolating critical resources.

The bulkhead pattern creates isolated pools of resources to contain failures and prevent cascade effects.

**Implementation**: Create separate thread pools, connection pools, and memory limits for different types of operations. Use semaphores to limit concurrent operations and prevent resource exhaustion in any single component.

**Example Implementation**:

```python
import asyncio
import aiohttp
from asyncio import Semaphore
from typing import Dict

class BulkheadPattern:
    def __init__(self):
        # Separate connection pools for different services
        self.pools: Dict[str, aiohttp.ClientSession] = {}
        self.semaphores: Dict[str, Semaphore] = {}
        
        # Configure different limits for different operations
        self.limits = {
            'database': 10,
            'external_api': 5,
            'file_operations': 3
        }
    
    async def get_session(self, service: str) -> aiohttp.ClientSession:
        if service not in self.pools:
            # Create isolated connection pool for this service
            connector = aiohttp.TCPConnector(
                limit=self.limits.get(service, 10),
                limit_per_host=self.limits.get(service, 10)
            )
            self.pools[service] = aiohttp.ClientSession(connector=connector)
            self.semaphores[service] = Semaphore(self.limits.get(service, 10))
        
        return self.pools[service]
    
    async def execute_with_bulkhead(self, service: str, operation):
        """Execute operation with bulkhead isolation"""
        semaphore = self.semaphores[service]
        
        async with semaphore:
            return await operation()

# Usage example
bulkhead = BulkheadPattern()

async def call_external_api():
    session = await bulkhead.get_session('external_api')
    
    async def operation():
        async with session.get('https://api.example.com/data') as response:
            return await response.json()
    
    return await bulkhead.execute_with_bulkhead('external_api', operation)
```

**8. Idempotency and Safe Retries**
**Priority: Production** | **Why Essential**: Prevents data corruption and enables safe retries in distributed systems with duplicate operations.

**Implementation**: Use idempotency keys for all state-changing operations. Implement deterministic key generation and maintain a registry of processed operations to prevent duplicate processing.

**Example Implementation**:

```python
import hashlib
import json
import redis
from typing import Any, Dict

class IdempotencyHandler:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.ttl_seconds = 86400  # 24 hours
    
    def generate_idempotency_key(self, operation: str, data: Dict[str, Any]) -> str:
        """Generate deterministic idempotency key"""
        content = f"{operation}:{json.dumps(data, sort_keys=True)}"
        return hashlib.sha256(content.encode()).hexdigest()
    
    def is_processed(self, key: str) -> bool:
        """Check if operation was already processed"""
        return self.redis.exists(f"idempotency:{key}")
    
    def mark_processed(self, key: str, result: Any):
        """Mark operation as processed with result"""
        self.redis.setex(
            f"idempotency:{key}",
            self.ttl_seconds,
            json.dumps(result)
        )
    
    def get_previous_result(self, key: str) -> Any:
        """Get result from previous processing"""
        result = self.redis.get(f"idempotency:{key}")
        return json.loads(result) if result else None

# Usage example
idempotency_handler = IdempotencyHandler(redis_client)

def create_user(user_data: Dict[str, Any]):
    key = idempotency_handler.generate_idempotency_key("create_user", user_data)
    
    if idempotency_handler.is_processed(key):
        return idempotency_handler.get_previous_result(key)
    
    # Process the operation
    result = {"user_id": "123", "status": "created"}
    
    # Mark as processed
    idempotency_handler.mark_processed(key, result)
    
    return result
```

**9. Structured Logging and Observability**
**Priority: Production** | **Why Essential**: Enables effective debugging, monitoring, and incident response in distributed systems.

**Implementation**: Use structured logging with consistent field names, request tracing, and correlation IDs. Implement comprehensive metrics collection and distributed tracing for end-to-end visibility.

**Example Implementation**:

```python
import logging
import uuid
from contextvars import ContextVar
from typing import Dict, Any

# Request context for correlation IDs
request_id: ContextVar[str] = ContextVar('request_id', default=None)

class StructuredLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
        self.setup_logging()
    
    def setup_logging(self):
        """Setup structured logging format"""
        formatter = logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - '
            '%(request_id)s - %(message)s'
        )
        
        handler = logging.StreamHandler()
        handler.setFormatter(formatter)
        self.logger.addHandler(handler)
    
    def log(self, level: str, message: str, **kwargs):
        """Log with structured data"""
        extra = {
            'request_id': request_id.get(),
            **kwargs
        }
        
        getattr(self.logger, level.lower())(message, extra=extra)

# Usage example
logger = StructuredLogger('my_app')

def process_request(user_id: str, data: Dict[str, Any]):
    # Set request ID for correlation
    req_id = str(uuid.uuid4())
    request_id.set(req_id)
    
    logger.log('info', 'Processing request', 
               user_id=user_id, 
               data_size=len(data))
    
    # Process request...
    
    logger.log('info', 'Request completed', 
               user_id=user_id, 
               status='success')
```

**10. Asynchronous Processing & Message Queues**
**Priority: Production** | **Why Essential**: Decouples components, provides resilience through persistent queues, and enables horizontal scaling.

Async processing prevents blocking operations from affecting user experience and provides natural resilience through message persistence.

**Implementation**: Use message queues with dead letter exchanges, priority queues, and comprehensive retry logic. Implement task routing based on priority and implement delayed task processing for time-sensitive operations.

**Example Implementation**:

```python
import asyncio
import aio_pika
from typing import Any, Dict
import json

class AsyncMessageProcessor:
    def __init__(self, rabbitmq_url: str):
        self.rabbitmq_url = rabbitmq_url
        self.connection = None
        self.channel = None
    
    async def connect(self):
        """Connect to RabbitMQ"""
        self.connection = await aio_pika.connect_robust(self.rabbitmq_url)
        self.channel = await self.connection.channel()
        
        # Declare queues with dead letter exchange
        await self.channel.declare_queue(
            'tasks',
            durable=True,
            arguments={
                'x-dead-letter-exchange': 'dlx',
                'x-dead-letter-routing-key': 'failed'
            }
        )
        
        await self.channel.declare_queue(
            'failed_tasks',
            durable=True
        )
    
    async def publish_task(self, task_data: Dict[str, Any], priority: int = 0):
        """Publish task to queue"""
        message = aio_pika.Message(
            body=json.dumps(task_data).encode(),
            delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            priority=priority
        )
        
        await self.channel.default_exchange.publish(
            message,
            routing_key='tasks'
        )
    
    async def process_tasks(self, processor_func):
        """Process tasks with retry logic"""
        async def process_message(message):
            async with message.process():
                try:
                    task_data = json.loads(message.body.decode())
                    await processor_func(task_data)
                except Exception as e:
                    # Reject message to dead letter queue
                    await message.reject(requeue=False)
                    raise e
        
        queue = await self.channel.declare_queue('tasks')
        await queue.consume(process_message)

# Usage example
async def process_user_task(task_data: Dict[str, Any]):
    # Process the task
    user_id = task_data['user_id']
    # ... processing logic
    pass

processor = AsyncMessageProcessor('amqp://localhost')
await processor.connect()
await processor.process_tasks(process_user_task)
```

**11. Caching Patterns**
**Priority: Production** | **Why Essential**: Reduces load on backends, improves performance, and provides fallback data during outages.

Effective caching is crucial for performance and can serve as a resilience mechanism when primary data sources fail.

**Implementation**: Implement multi-level caching (L1 local, L2 distributed) with stale-while-revalidate patterns. Use cache-aside patterns and maintain stale cache copies for fallback during backend failures.

**Example Implementation**:

```python
import redis
import asyncio
from typing import Any, Optional
import time

class MultiLevelCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.local_cache = {}  # L1 cache
        self.local_cache_ttl = 300  # 5 minutes
    
    async def get(self, key: str, fetch_func=None) -> Optional[Any]:
        """Get value with multi-level cache"""
        # Try L1 cache first
        if key in self.local_cache:
            value, timestamp = self.local_cache[key]
            if time.time() - timestamp < self.local_cache_ttl:
                return value
        
        # Try L2 cache (Redis)
        cached_value = self.redis.get(key)
        if cached_value:
            value = json.loads(cached_value)
            # Update L1 cache
            self.local_cache[key] = (value, time.time())
            return value
        
        # Fetch from source
        if fetch_func:
            try:
                value = await fetch_func()
                # Cache in both levels
                self.redis.setex(key, 3600, json.dumps(value))  # 1 hour TTL
                self.local_cache[key] = (value, time.time())
                return value
            except Exception:
                # Return stale cache if available
                stale_value = self.redis.get(f"{key}:stale")
                if stale_value:
                    return json.loads(stale_value)
                raise
        
        return None
    
    def set_stale_copy(self, key: str, value: Any):
        """Set stale copy for fallback"""
        self.redis.setex(f"{key}:stale", 86400, json.dumps(value))  # 24 hours

# Usage example
cache = MultiLevelCache(redis_client)

async def get_user_data(user_id: str):
    async def fetch_user():
        # Fetch from database
        return {"user_id": user_id, "name": "John Doe"}
    
    return await cache.get(f"user:{user_id}", fetch_user)
```

## Conclusion

Architecting downtime-tolerant applications in Kubernetes requires a holistic approach combining Kubernetes primitives, application design patterns, comprehensive testing, and operational excellence. The journey from basic availability to advanced resilience is iterative—start with fundamental practices like multiple replicas and health checks, then progressively add sophistication through chaos engineering, service mesh, and multi-region deployments.

Key takeaways for cloud-native engineers and SREs:

1. **Start Simple**: Begin with the minimal resilient setup and iterate based on requirements
2. **Test Continuously**: Implement chaos engineering from day one, not as an afterthought
3. **Monitor Everything**: Comprehensive observability is essential for maintaining resilience
4. **Automate Recovery**: Reduce mean time to recovery through automation and self-healing systems
5. **Balance Cost and Resilience**: Use cloud-native features like spot instances wisely
6. **Learn from Failures**: Every incident is an opportunity to improve system resilience

The investment in downtime tolerance pays dividends through improved customer satisfaction, reduced operational burden, and compliance with stringent SLAs. By following these practices and continuously testing your assumptions, you'll build systems that gracefully handle the inevitable failures in distributed cloud environments.

Remember: Resilience is not a destination but a continuous journey of improvement, testing, and adaptation to new failure modes and requirements.

## References and Further Reading

- [Kubernetes Documentation: Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes Documentation: Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes Documentation: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes Documentation: Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Resilience4j Documentation](https://resilience4j.readme.io/) - Circuit breaker, retry, and bulkhead patterns
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html) - Martin Fowler's explanation
- [Chaos Engineering: Building Confidence in System Behavior](https://www.oreilly.com/library/view/chaos-engineering/9781491988459/)
- [Prometheus Monitoring Best Practices](https://prometheus.io/docs/practices/)
- [Jaeger Distributed Tracing](https://www.jaegertracing.io/docs/)
- [OpenTelemetry](https://opentelemetry.io/docs/) - Observability framework
- [CNCF Cloud Native Trail Map](https://github.com/cncf/trailmap)
- [Google SRE Books](https://sre.google/books/) - Site Reliability Engineering
- [AWS Well-Architected Framework: Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes/) - Security guidelines
- [Kubernetes Failure Stories](https://k8s.af/) - Learn from real-world incidents