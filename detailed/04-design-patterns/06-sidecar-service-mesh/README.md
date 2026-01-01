# Sidecar & Service Mesh

The Sidecar pattern deploys helper containers alongside application containers to handle cross-cutting concerns. A Service Mesh extends this to manage all service-to-service communication.

## Table of Contents
1. [Sidecar Pattern](#sidecar-pattern)
2. [Service Mesh](#service-mesh)
3. [Popular Solutions](#popular-solutions)
4. [When to Use](#when-to-use)
5. [Interview Questions](#interview-questions)

---

## Sidecar Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    Sidecar Pattern                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Problem: Cross-cutting concerns in every service          │
│                                                              │
│  Without Sidecar:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Every service needs:                                   │ │
│  │  • Logging                                              │ │
│  │  • Metrics                                              │ │
│  │  • Service discovery                                    │ │
│  │  • Circuit breaking                                     │ │
│  │  • TLS/mTLS                                             │ │
│  │  • Retries                                              │ │
│  │                                                          │ │
│  │  Result: Duplicated code, inconsistent implementations │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  With Sidecar:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                        Pod                              │ │
│  │  ┌─────────────────┐    ┌─────────────────────────┐   │ │
│  │  │   Application   │    │        Sidecar           │   │ │
│  │  │   Container     │◀──▶│       (Envoy/etc)        │   │ │
│  │  │                 │    │                          │   │ │
│  │  │  Just business  │    │ • Logging                │   │ │
│  │  │  logic!         │    │ • Metrics                │   │ │
│  │  │                 │    │ • mTLS                   │   │ │
│  │  │                 │    │ • Service discovery      │   │ │
│  │  │                 │    │ • Circuit breaking       │   │ │
│  │  └─────────────────┘    └─────────────────────────┘   │ │
│  │                                                         │ │
│  │  Sidecar handles networking, app is simpler           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### How Sidecar Works

```
┌─────────────────────────────────────────────────────────────┐
│                  How Sidecar Works                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Traffic Flow:                                               │
│                                                              │
│       Incoming                          Outgoing             │
│       Request                           Request              │
│          │                                 ▲                 │
│          ▼                                 │                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                         Pod                            │  │
│  │    ┌─────────────┐              ┌─────────────┐       │  │
│  │    │   Sidecar   │◀─localhost──▶│    App      │       │  │
│  │    │   Proxy     │              │  Container  │       │  │
│  │    └──────┬──────┘              └─────────────┘       │  │
│  │           │                                            │  │
│  └───────────│────────────────────────────────────────────┘  │
│              │                                               │
│              ▼                                               │
│      Other Services                                          │
│                                                              │
│  Sidecar intercepts all network traffic:                   │
│  • Inbound: Sidecar → App                                  │
│  • Outbound: App → Sidecar → Other services               │
│                                                              │
│  App communicates to localhost, sidecar handles the rest   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Common Sidecar Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│              Common Sidecar Use Cases                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Proxy Sidecar                                           │
│     • Envoy, NGINX, HAProxy                                │
│     • Load balancing, circuit breaking, retries            │
│                                                              │
│  2. Logging Sidecar                                         │
│     • Fluentd, Fluent Bit                                  │
│     • Collect and forward logs                             │
│                                                              │
│  3. Monitoring Sidecar                                      │
│     • Prometheus exporter                                   │
│     • Collect and expose metrics                           │
│                                                              │
│  4. Security Sidecar                                        │
│     • TLS termination                                       │
│     • Authentication/Authorization                         │
│                                                              │
│  5. Config Sidecar                                          │
│     • Watch config changes                                  │
│     • Update app configuration                             │
│                                                              │
│  6. Service Discovery Sidecar                              │
│     • Register with service registry                       │
│     • Health checks                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Service Mesh

```
┌─────────────────────────────────────────────────────────────┐
│                      Service Mesh                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Service Mesh = Sidecar proxies + Control Plane            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Control Plane                        │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │  • Configuration management                       │ │ │
│  │  │  • Certificate authority (mTLS)                   │ │ │
│  │  │  • Policy enforcement                             │ │ │
│  │  │  • Telemetry aggregation                         │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │         │              │              │                 │ │
│  │         ▼              ▼              ▼                 │ │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐          │ │
│  │  │  Service  │  │  Service  │  │  Service  │          │ │
│  │  │  A + Proxy│  │  B + Proxy│  │  C + Proxy│          │ │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘          │ │
│  │        │              │              │                 │ │
│  │        └──────────────┴──────────────┘                 │ │
│  │              Data Plane (Proxy-to-Proxy)               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Data Plane: Handles actual traffic between services       │
│  Control Plane: Configures and manages the data plane      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Service Mesh Features

```
┌─────────────────────────────────────────────────────────────┐
│                 Service Mesh Features                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Traffic Management                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Load balancing (round-robin, least-conn, etc.)     │ │
│  │  • Traffic splitting (canary, A/B testing)            │ │
│  │  • Retries and timeouts                               │ │
│  │  • Circuit breaking                                    │ │
│  │  • Rate limiting                                       │ │
│  │                                                         │ │
│  │  Example - Canary deployment:                          │ │
│  │  ┌──────────┐   90%   ┌──────────┐                    │ │
│  │  │ Incoming │────────▶│ V1 (old) │                    │ │
│  │  │ Traffic  │         └──────────┘                    │ │
│  │  │          │   10%   ┌──────────┐                    │ │
│  │  │          │────────▶│ V2 (new) │                    │ │
│  │  └──────────┘         └──────────┘                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  2. Security (mTLS)                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  All service-to-service traffic encrypted             │ │
│  │                                                         │ │
│  │  ┌──────────┐   mTLS    ┌──────────┐                  │ │
│  │  │ Service  │◀─────────▶│ Service  │                  │ │
│  │  │ A Proxy  │  Encrypted│ B Proxy  │                  │ │
│  │  └──────────┘  & Authed │──────────┘                  │ │
│  │                                                         │ │
│  │  • Automatic certificate rotation                     │ │
│  │  • Service identity verification                      │ │
│  │  • Zero-trust security                                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  3. Observability                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Distributed tracing (automatic span creation)      │ │
│  │  • Metrics (latency, errors, requests/sec)           │ │
│  │  • Access logs                                         │ │
│  │  • Service dependency graphs                          │ │
│  │                                                         │ │
│  │  No code changes required!                             │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  4. Policy Enforcement                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Authorization policies                              │ │
│  │  • Rate limits                                         │ │
│  │  • Access control                                      │ │
│  │                                                         │ │
│  │  "Service A can call Service B, but not C"            │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Popular Solutions

```
┌─────────────────────────────────────────────────────────────┐
│                   Popular Solutions                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Istio (Most Popular)                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Data Plane: Envoy proxy                               │ │
│  │  Control Plane: istiod                                 │ │
│  │                                                         │ │
│  │  ✓ Feature-rich                                        │ │
│  │  ✓ Large community                                     │ │
│  │  ✗ Complex, resource-intensive                        │ │
│  │  ✗ Steep learning curve                               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Linkerd                                                    │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Data Plane: linkerd2-proxy (Rust)                     │ │
│  │  Control Plane: linkerd                                │ │
│  │                                                         │ │
│  │  ✓ Lightweight, fast                                   │ │
│  │  ✓ Simple to operate                                   │ │
│  │  ✓ CNCF graduated                                      │ │
│  │  ✗ Fewer features than Istio                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Envoy (Proxy, not full mesh)                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • High-performance L7 proxy                           │ │
│  │  • Can be used standalone or with Istio               │ │
│  │  • xDS API for dynamic configuration                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  AWS App Mesh                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  • Managed service mesh for AWS                        │ │
│  │  • Uses Envoy                                          │ │
│  │  • Integrates with AWS services                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Comparison:                                                │
│  ┌────────────┬────────────┬────────────┬────────────────┐ │
│  │            │   Istio    │  Linkerd   │   Consul       │ │
│  ├────────────┼────────────┼────────────┼────────────────┤ │
│  │ Complexity │   High     │   Low      │   Medium       │ │
│  │ Performance│   Good     │   Best     │   Good         │ │
│  │ Features   │   Most     │   Core     │   Many         │ │
│  │ Resources  │   Heavy    │   Light    │   Medium       │ │
│  └────────────┴────────────┴────────────┴────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## When to Use

```
┌─────────────────────────────────────────────────────────────┐
│                    When to Use                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Use Service Mesh When:                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✓ Many microservices (10+)                             │ │
│  │ ✓ Need zero-trust security                             │ │
│  │ ✓ Complex traffic management (canary, A/B)            │ │
│  │ ✓ Want observability without code changes             │ │
│  │ ✓ Polyglot services (different languages)             │ │
│  │ ✓ Have platform/DevOps team to operate                │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Don't Use When:                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ ✗ Small number of services (<10)                       │ │
│  │ ✗ Simple communication patterns                        │ │
│  │ ✗ No Kubernetes                                        │ │
│  │ ✗ Limited operational expertise                       │ │
│  │ ✗ Latency-critical with no room for proxy overhead    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Alternatives:                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Library-based approach (Resilience4j, etc.)         │ │
│  │   Good for small-medium deployments                   │ │
│  │                                                         │ │
│  │ • API Gateway only                                     │ │
│  │   Good for north-south traffic focus                  │ │
│  │                                                         │ │
│  │ • Cloud provider solutions                             │ │
│  │   AWS App Mesh, GCP Traffic Director                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Overhead Considerations:                                   │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ • Latency: +1-2ms per hop (proxy processing)          │ │
│  │ • Memory: 50-100MB per sidecar                        │ │
│  │ • CPU: 0.1-0.5 vCPU per sidecar                       │ │
│  │ • Control plane resources                              │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

### Basic
1. What is the sidecar pattern?
2. What is a service mesh?

### Intermediate
3. What are the benefits of using a service mesh?
4. Compare Istio vs Linkerd.

### Advanced
5. How would you implement mTLS without a service mesh?
6. When would you choose a library-based approach over a service mesh?

---

[← Previous: Circuit Breaker](../05-circuit-breaker/README.md) | [Back to Design Patterns](../README.md)
