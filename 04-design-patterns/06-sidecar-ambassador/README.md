# Sidecar & Ambassador Patterns

These patterns extract cross-cutting concerns from application code into separate processes, commonly used in service mesh architectures.

## Table of Contents
- [Sidecar Pattern](#sidecar-pattern)
- [Ambassador Pattern](#ambassador-pattern)
- [Service Mesh](#service-mesh)
- [Implementation](#implementation)
- [Key Takeaways](#key-takeaways)

## Sidecar Pattern

A sidecar is a helper container that runs alongside your main application, providing supporting features.

```
┌────────────────────────────────────────────────────────┐
│                        Pod                             │
│  ┌────────────────────────────────────────────────┐   │
│  │                                                │   │
│  │  ┌─────────────────┐    ┌─────────────────┐   │   │
│  │  │   Application   │    │    Sidecar      │   │   │
│  │  │                 │◀──▶│                 │   │   │
│  │  │  - Business     │    │  - Logging      │   │   │
│  │  │    logic        │    │  - Metrics      │   │   │
│  │  │                 │    │  - Proxy        │   │   │
│  │  └─────────────────┘    └─────────────────┘   │   │
│  │                                                │   │
│  │           localhost communication              │   │
│  └────────────────────────────────────────────────┘   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Use Cases

| Use Case | Description |
|----------|-------------|
| Logging | Collect and ship logs (Fluentd) |
| Monitoring | Export metrics (Prometheus exporter) |
| Proxy | Handle network traffic (Envoy) |
| Configuration | Dynamic config updates |
| Security | TLS termination, auth |
| Synchronization | Data sync, caching |

### Benefits

```
┌────────────────────────────────────────────────────────┐
│                Sidecar Benefits                        │
│                                                        │
│  1. Separation of concerns                            │
│     App focuses on business logic                     │
│     Sidecar handles infrastructure                    │
│                                                        │
│  2. Language agnostic                                 │
│     Works with any language/framework                 │
│     Same sidecar for Go, Java, Python apps           │
│                                                        │
│  3. Independent lifecycle                             │
│     Update sidecar without changing app              │
│     Different release cycles                          │
│                                                        │
│  4. Shared fate                                       │
│     Runs on same host as app                         │
│     Shares lifecycle (starts/stops together)         │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Logging Sidecar Example

```yaml
# Kubernetes Pod with logging sidecar
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  # Main application
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  # Logging sidecar
  - name: log-shipper
    image: fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluentd-config
      mountPath: /fluentd/etc

  volumes:
  - name: logs
    emptyDir: {}
  - name: fluentd-config
    configMap:
      name: fluentd-config
```

## Ambassador Pattern

An ambassador handles outgoing connections on behalf of the application.

```
┌────────────────────────────────────────────────────────┐
│                     Ambassador                         │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │                      Pod                          │ │
│  │  ┌─────────────┐      ┌─────────────┐            │ │
│  │  │ Application │──────│ Ambassador  │            │ │
│  │  │             │      │             │            │ │
│  │  │ calls       │      │ - Retries   │            │ │
│  │  │ localhost   │      │ - Circuit   │──────────▶ │ │
│  │  │             │      │   breaker   │   External │ │
│  │  │             │      │ - Logging   │   Services │ │
│  │  └─────────────┘      └─────────────┘            │ │
│  │                                                   │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Use Cases

| Use Case | Description |
|----------|-------------|
| Connection pooling | Manage database connections |
| Retry logic | Retry failed requests |
| Circuit breaking | Fail fast on errors |
| Rate limiting | Limit outbound requests |
| Protocol translation | HTTP to gRPC |
| Service discovery | Resolve service endpoints |

### Ambassador vs Sidecar

```
Sidecar:
- General-purpose helper
- Handles various concerns
- May handle inbound traffic

Ambassador:
- Specifically for outbound connections
- Proxy to external services
- Offload network complexity
```

## Service Mesh

A service mesh uses sidecars to handle all service-to-service communication.

```
┌────────────────────────────────────────────────────────┐
│                    Service Mesh                        │
│                                                        │
│  ┌───────────────────────────────────────────────────┐│
│  │                 Control Plane                      ││
│  │  ┌────────────────────────────────────────────┐   ││
│  │  │  Config, Policy, Certificates, Discovery   │   ││
│  │  └────────────────────────────────────────────┘   ││
│  └───────────────────────────────────────────────────┘│
│                           │                            │
│           ┌───────────────┴───────────────┐           │
│           ▼                               ▼           │
│  ┌─────────────────┐           ┌─────────────────┐   │
│  │ ┌─────┐ ┌─────┐ │           │ ┌─────┐ ┌─────┐ │   │
│  │ │ App │◀▶│Proxy│◀├───────────┤▶│Proxy│◀▶│ App │ │   │
│  │ └─────┘ └─────┘ │  (mTLS)   │ └─────┘ └─────┘ │   │
│  │    Service A    │           │    Service B    │   │
│  └─────────────────┘           └─────────────────┘   │
│                                                        │
│  Data Plane (Envoy proxies)                           │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Service Mesh Features

```
┌────────────────────────────────────────────────────────┐
│              Service Mesh Capabilities                 │
│                                                        │
│  Traffic Management:                                   │
│  - Load balancing                                     │
│  - Traffic splitting (canary, A/B)                   │
│  - Retries, timeouts, circuit breaking               │
│                                                        │
│  Security:                                             │
│  - Mutual TLS (mTLS)                                 │
│  - Authorization policies                             │
│  - Certificate management                             │
│                                                        │
│  Observability:                                        │
│  - Distributed tracing                               │
│  - Metrics collection                                 │
│  - Access logging                                     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Popular Service Meshes

| Mesh | Proxy | Notes |
|------|-------|-------|
| Istio | Envoy | Feature-rich, complex |
| Linkerd | linkerd2-proxy | Lightweight, simple |
| Consul Connect | Envoy | HashiCorp ecosystem |
| AWS App Mesh | Envoy | AWS native |

### Istio Traffic Management

```yaml
# Virtual Service for traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10

# Destination Rule for circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## Implementation

### Envoy as Sidecar

```yaml
# Envoy configuration
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8080
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: local_app
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: local_app
    connect_timeout: 5s
    type: STATIC
    load_assignment:
      cluster_name: local_app
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 3000
```

### Application Deployment

```yaml
# Kubernetes deployment with Istio sidecar injection
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        sidecar.istio.io/inject: "true"  # Enable sidecar
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
```

## Trade-offs

### Pros

```
✓ Clean separation of concerns
✓ Consistent infrastructure across services
✓ Language-agnostic
✓ Centralized policy management
✓ Rich observability
```

### Cons

```
✗ Added complexity
✗ Resource overhead (CPU, memory)
✗ Latency overhead (~1ms per hop)
✗ Debugging can be harder
✗ Learning curve
```

### When to Use

```
Good fit:
- Large microservices deployments
- Polyglot environments
- Need for consistent security/observability
- Kubernetes-based infrastructure

Not necessary:
- Small number of services
- Monolithic applications
- Simple communication patterns
- Resource-constrained environments
```

## Key Takeaways

1. **Sidecar** extracts cross-cutting concerns from apps
2. **Ambassador** handles outbound connection complexity
3. **Service mesh** = sidecars + control plane
4. **Benefits**: consistency, language agnostic, separation
5. **Costs**: complexity, resource overhead, latency
6. **Start simple** - add mesh when you need it

## Practice Questions

1. When would you use a sidecar vs embedding functionality in the app?
2. How does a service mesh provide mTLS?
3. What's the latency overhead of a service mesh?
4. Design a logging strategy using sidecars.

## Further Reading

- [Istio Documentation](https://istio.io/latest/docs/)
- [Envoy Proxy](https://www.envoyproxy.io/)
- [Linkerd](https://linkerd.io/)
- [Service Mesh Patterns](https://www.oreilly.com/library/view/the-enterprise-path/9781492041795/)

---

[Back to Module 4](../README.md) | Next: [Module 5: System Designs](../../05-system-designs/README.md)
