# Cluster Management en Envoy

---

**Módulo**: 3 - Arquitectura de Envoy
**Tema**: Clusters, Load Balancing, Health Checks
**Tiempo estimado**: 3 horas
**Prerrequisitos**: [04_filter_chains.md](04_filter_chains.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás qué es un cluster y cómo se configura
- Conocerás los algoritmos de load balancing
- Comprenderás health checking y circuit breaking
- Sabrás cómo funciona el connection pooling

---

## 1. Conceptos Básicos

### 1.1 ¿Qué es un Cluster?

Un **cluster** es un grupo de endpoints (hosts) que pueden servir tráfico:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Cluster: "api-v1"                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Endpoints:                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │ 10.0.1.1:8080 │  │ 10.0.1.2:8080 │  │ 10.0.1.3:8080 │       │
│  │ weight: 1     │  │ weight: 1     │  │ weight: 1     │       │
│  │ healthy: ✓    │  │ healthy: ✓    │  │ healthy: ✗    │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
│                                                                 │
│  Config:                                                        │
│  ├── LB Policy: ROUND_ROBIN                                    │
│  ├── Connect Timeout: 5s                                        │
│  ├── Health Check: HTTP /health every 10s                       │
│  └── Circuit Breaker: max 100 connections                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Cluster Manager

```cpp
// source/common/upstream/cluster_manager_impl.h:98

class ClusterManagerImpl : public ClusterManager {
    // Mapa de clusters por nombre
    ClusterMap clusters_;

    // Thread-local cluster data
    ThreadLocal::TypedSlot<ThreadLocalClusterManagerImpl> tls_;

    // Config subscription (CDS)
    CdsApiPtr cds_api_;
};
```

---

## 2. Configuración de Clusters

### 2.1 Cluster Estático

```yaml
clusters:
  - name: api_cluster
    type: STATIC
    connect_timeout: 5s
    lb_policy: ROUND_ROBIN

    # Endpoints estáticos
    load_assignment:
      cluster_name: api_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.1.1
                    port_value: 8080
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.1.2
                    port_value: 8080

    # Health checks
    health_checks:
      - timeout: 1s
        interval: 10s
        healthy_threshold: 2
        unhealthy_threshold: 3
        http_health_check:
          path: "/health"
```

### 2.2 Tipos de Discovery

| Tipo           | Descripción                | Uso                          |
| -------------- | -------------------------- | ---------------------------- |
| `STATIC`       | Endpoints en config        | Dev, configs simples         |
| `STRICT_DNS`   | DNS lookup                 | Kubernetes headless services |
| `LOGICAL_DNS`  | DNS con cache              | External services            |
| `EDS`          | Endpoint Discovery Service | Production con xDS           |
| `ORIGINAL_DST` | IP destino original        | Transparent proxy            |

### 2.3 Cluster Dinámico (EDS)

```yaml
clusters:
  - name: dynamic_cluster
    type: EDS
    connect_timeout: 5s
    lb_policy: ROUND_ROBIN
    eds_cluster_config:
      service_name: my-service
      eds_config:
        api_config_source:
          api_type: GRPC
          grpc_services:
            - envoy_grpc:
                cluster_name: xds_cluster
```

---

## 3. Load Balancing

### 3.1 Algoritmos Disponibles

```cpp
// source/common/upstream/load_balancer_impl.cc

enum class LoadBalancerType {
    RoundRobin,      // Rotación circular
    LeastRequest,    // Menos requests activos
    Random,          // Aleatorio
    RingHash,        // Consistent hashing
    Maglev,          // Google Maglev
    ClusterProvided  // Custom del cluster
};
```

### 3.2 Round Robin

```cpp
// source/common/upstream/load_balancer_impl.cc:200-300

HostConstSharedPtr RoundRobinLoadBalancer::chooseHost(LoadBalancerContext*) {
    const auto& hosts = hostSet().healthyHosts();
    if (hosts.empty()) {
        return nullptr;
    }

    // Incrementar índice y wrap
    return hosts[rr_index_++ % hosts.size()];
}
```

```
Request 1 → Host A
Request 2 → Host B
Request 3 → Host C
Request 4 → Host A (wrap)
...
```

### 3.3 Least Request

```cpp
HostConstSharedPtr LeastRequestLoadBalancer::chooseHost(LoadBalancerContext*) {
    const auto& hosts = hostSet().healthyHosts();

    // Elegir N hosts aleatorios
    auto candidates = chooseRandomHosts(hosts, choice_count_);

    // Seleccionar el que tiene menos requests activos
    return *std::min_element(candidates.begin(), candidates.end(),
        [](const auto& a, const auto& b) {
            return a->stats().rq_active_.value() < b->stats().rq_active_.value();
        });
}
```

### 3.4 Ring Hash (Consistent Hashing)

Útil para sticky sessions y caching:

```yaml
clusters:
  - name: cache_cluster
    lb_policy: RING_HASH
    ring_hash_lb_config:
      minimum_ring_size: 1024
      maximum_ring_size: 8388608
      hash_function: XX_HASH
```

```cpp
// El hash se calcula sobre un atributo del request
// Por ejemplo, header o cookie

HostConstSharedPtr RingHashLoadBalancer::chooseHost(LoadBalancerContext* ctx) {
    uint64_t hash = ctx->computeHashKey();  // Hash del request
    return ring_.chooseHost(hash);           // Buscar en el ring
}
```

```
Ring:
[0    ....    Host_A    ....    Host_B    ....    Host_C    ....    MAX]
              ^
              Hash(request) cae aquí → Host_A
```

### 3.5 Configuración de Hash

```yaml
route_config:
  virtual_hosts:
    - routes:
        - match:
            prefix: "/"
          route:
            cluster: cache_cluster
            hash_policy:
              # Hash por header
              - header:
                  header_name: "x-user-id"
              # O por cookie
              - cookie:
                  name: "session"
                  ttl: 3600s
              # O por IP
              - connection_properties:
                  source_ip: true
```

---

## 4. Health Checking

### 4.1 Tipos de Health Checks

```yaml
health_checks:
  # HTTP health check
  - timeout: 1s
    interval: 10s
    healthy_threshold: 2
    unhealthy_threshold: 3
    http_health_check:
      path: "/health"
      expected_statuses:
        - start: 200
          end: 299

  # TCP health check
  - timeout: 1s
    interval: 10s
    tcp_health_check:
      send:
        text: "ping"
      receive:
        - text: "pong"

  # gRPC health check
  - timeout: 1s
    interval: 10s
    grpc_health_check:
      service_name: "myservice"
```

### 4.2 Código de Health Checker

```cpp
// source/common/upstream/health_checker_impl.cc:300-600

void HttpHealthCheckerImpl::onResponseComplete() {
    if (response_code_ >= 200 && response_code_ < 300) {
        handleSuccess();
    } else {
        handleFailure(FailureType::Active);
    }
}

void HealthCheckerImpl::handleSuccess() {
    consecutive_failures_ = 0;
    consecutive_successes_++;

    if (consecutive_successes_ >= healthy_threshold_) {
        // Marcar como healthy
        host_->healthFlagClear(Host::HealthFlag::FAILED_ACTIVE_HC);
    }
}
```

### 4.3 Outlier Detection (Passive Health)

Detecta hosts problemáticos basándose en tráfico real:

```yaml
clusters:
  - name: api_cluster
    outlier_detection:
      # Eyectar después de N errores consecutivos
      consecutive_5xx: 5

      # Tiempo mínimo de eyección
      base_ejection_time: 30s

      # Máximo % de hosts eyectados
      max_ejection_percent: 50

      # Eyectar por errores de gateway
      consecutive_gateway_failure: 5

      # Intervalo de análisis
      interval: 10s
```

---

## 5. Circuit Breaking

### 5.1 Concepto

Protege backends de sobrecarga cortando tráfico cuando se exceden límites:

```
                    Normal                     Circuit Open
                    ┌─────┐                    ┌─────┐
Requests ──────────>│     │──────────>         │     │───X──> (503)
                    │ CB  │  Backend           │ CB  │
                    └─────┘                    └─────┘
                      │                          │
              connections < max         connections >= max
```

### 5.2 Configuración

```yaml
clusters:
  - name: api_cluster
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          # Máximo de conexiones TCP
          max_connections: 100

          # Máximo de requests pendientes
          max_pending_requests: 100

          # Máximo de requests concurrentes
          max_requests: 1000

          # Máximo de retries simultáneos
          max_retries: 3

        - priority: HIGH
          max_connections: 200
          max_requests: 2000
```

### 5.3 Código

```cpp
// source/common/upstream/resource_manager_impl.cc:50-150

bool ResourceManager::canCreate() {
    // Verificar si se puede crear nueva conexión
    return current_connections_ < max_connections_;
}

void ClusterCircuitBreakerStats::checkTresholds() {
    if (pending_requests_ >= max_pending_requests_) {
        // Overflow - rechazar request
        stats_.upstream_rq_pending_overflow_.inc();
        return false;
    }
    return true;
}
```

---

## 6. Connection Pooling

### 6.1 HTTP/1.1 Pool

```
┌─────────────────────────────────────────────────────────────────┐
│                  HTTP/1.1 Connection Pool                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Host: 10.0.1.1:8080                                           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │ Connection 1│ │ Connection 2│ │ Connection 3│              │
│  │ (busy)      │ │ (idle)      │ │ (busy)      │              │
│  └─────────────┘ └─────────────┘ └─────────────┘              │
│                                                                 │
│  - 1 request por conexión (no multiplex)                       │
│  - Keep-alive para reutilizar                                   │
│  - max_connections limita total                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 HTTP/2 Pool

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP/2 Connection Pool                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Host: 10.0.1.1:8080                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Connection 1                                             │   │
│  │ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │   │
│  │ │Stream 1│ │Stream 3│ │Stream 5│ │Stream 7│            │   │
│  │ └────────┘ └────────┘ └────────┘ └────────┘            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  - Múltiples streams por conexión                               │
│  - max_concurrent_streams por conexión                          │
│  - Típicamente menos conexiones necesarias                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 Configuración de Pool

```yaml
clusters:
  - name: api_cluster
    http2_protocol_options:
      max_concurrent_streams: 100

    # Opciones de conexión upstream
    upstream_connection_options:
      tcp_keepalive:
        keepalive_probes: 3
        keepalive_time: 60
        keepalive_interval: 10
```

---

## 7. Prioridades y Localidades

### 7.1 Priority Levels

```yaml
clusters:
  - name: api_cluster
    load_assignment:
      cluster_name: api_cluster
      endpoints:
        # Priority 0 (highest) - Local datacenter
        - priority: 0
          lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.1.1
                    port_value: 8080

        # Priority 1 - Remote datacenter
        - priority: 1
          lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.2.1
                    port_value: 8080
```

### 7.2 Locality Aware Routing

```yaml
clusters:
  - name: api_cluster
    common_lb_config:
      locality_weighted_lb_config: {}
    load_assignment:
      endpoints:
        - locality:
            region: us-east-1
            zone: us-east-1a
          lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.1.1
                    port_value: 8080
          load_balancing_weight: 100

        - locality:
            region: us-west-2
            zone: us-west-2a
          lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 10.0.2.1
                    port_value: 8080
          load_balancing_weight: 50
```

---

## 8. Visualizar Clusters

```bash
# Ver todos los clusters
curl localhost:9901/clusters

# Output ejemplo:
# api_cluster::default_priority::max_connections::100
# api_cluster::default_priority::max_pending_requests::100
# api_cluster::10.0.1.1:8080::cx_active::5
# api_cluster::10.0.1.1:8080::rq_active::10
# api_cluster::10.0.1.1:8080::health_flags::healthy

# Ver stats de upstream
curl localhost:9901/stats | grep upstream
```

---

## 9. Autoevaluación

1. ¿Qué es un cluster y qué contiene?
2. ¿Cuándo usarías Ring Hash vs Round Robin?
3. ¿Cuál es la diferencia entre active y passive health checking?
4. ¿Qué problema resuelve circuit breaking?
5. ¿Por qué HTTP/2 necesita menos conexiones que HTTP/1.1?

---

**Siguiente**: [06_xds_protocol.md](06_xds_protocol.md) - xDS Protocol
