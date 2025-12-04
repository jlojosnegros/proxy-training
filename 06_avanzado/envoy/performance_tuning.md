# Performance Tuning en Envoy

---

**Módulo**: 6 - Avanzado (Envoy)
**Tema**: Optimización de rendimiento
**Tiempo estimado**: 3 horas
**Prerrequisitos**: Módulo 3 completo

---

## Objetivos de Aprendizaje

Al completar este documento:

- Conocerás las métricas clave de performance
- Entenderás cómo usar profiling (pprof)
- Sabrás configurar Envoy para alto throughput
- Podrás identificar y resolver bottlenecks

---

## 1. Métricas de Performance

### 1.1 Métricas Clave

```
┌─────────────────────────────────────────────────────────────────┐
│                    Key Performance Metrics                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LATENCY                                                       │
│  ─────────                                                      │
│  • P50, P99, P999 request latency                              │
│  • upstream_rq_time histogram                                  │
│  • downstream_rq_time histogram                                │
│                                                                 │
│  THROUGHPUT                                                    │
│  ───────────                                                    │
│  • Requests per second (RPS)                                   │
│  • Bytes transferred per second                                │
│  • downstream_cx_active (concurrent connections)               │
│                                                                 │
│  RESOURCE USAGE                                                │
│  ───────────────                                                │
│  • Memory: heap, thread local storage                          │
│  • CPU: per worker thread                                      │
│  • File descriptors: open connections                          │
│                                                                 │
│  ERROR RATES                                                   │
│  ────────────                                                   │
│  • upstream_rq_retry, upstream_rq_timeout                      │
│  • Circuit breaker triggers                                    │
│  • Connection failures                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Acceder a Stats

```bash
# Admin interface
curl http://localhost:15000/stats

# Solo counters
curl http://localhost:15000/stats?filter=counter

# Filtrar por prefijo
curl http://localhost:15000/stats?filter=upstream_rq

# Formato Prometheus
curl http://localhost:15000/stats/prometheus

# Stats específicos útiles
curl http://localhost:15000/stats | grep -E "downstream_cx|upstream_rq|memory"
```

---

## 2. Profiling con pprof

### 2.1 Habilitar pprof

```yaml
# envoy.yaml
admin:
  access_log_path: /dev/null
  profile_path: /var/log/envoy/
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 15000
```

### 2.2 Capturar CPU Profile

```bash
# Iniciar profiling por 30 segundos
curl -X POST "http://localhost:15000/cpuprofiler?enable=y"
sleep 30
curl -X POST "http://localhost:15000/cpuprofiler?enable=n"

# El archivo se guarda en profile_path
ls /var/log/envoy/*.prof

# Analizar con pprof
# Requiere Go pprof tools
go tool pprof -http=:8080 /var/log/envoy/envoy.prof
```

### 2.3 Heap Profile

```bash
# Obtener heap profile actual
curl "http://localhost:15000/heapprofiler?enable=y"

# Descargar heap dump
curl -o heap.prof "http://localhost:15000/heap"

# Analizar
go tool pprof -http=:8080 heap.prof
```

### 2.4 Interpretar Resultados

```
┌─────────────────────────────────────────────────────────────────┐
│                   pprof Interpretation                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU Profile - Qué buscar:                                     │
│  ─────────────────────────                                      │
│  • Funciones con alto % de tiempo                              │
│  • Hot paths en filter chain                                   │
│  • Allocations en hot path                                     │
│  • Lock contention                                             │
│                                                                 │
│  Ejemplo de output:                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  40% - TcpProxy::onData                                  │  │
│  │  25% - Http::ConnectionManager::decode*                  │  │
│  │  15% - Upstream::TcpPoolImpl::*                         │  │
│  │  10% - SSL_read / SSL_write                             │  │
│  │  10% - Other                                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Heap Profile - Qué buscar:                                    │
│  ──────────────────────────                                     │
│  • Allocations que crecen con el tiempo → posible leak        │
│  • Grandes allocations en hot path                            │
│  • Buffer sizes                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Tuning de Workers

### 3.1 Número de Workers

```yaml
# Por defecto: número de cores
# Configurar manualmente si necesario

# Opción 1: CLI
# envoy --concurrency 4

# Opción 2: Bootstrap
bootstrap_extensions:
  - name: envoy.extensions.bootstrap.internal_listener
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.bootstrap.internal_listener.v3.InternalListener
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Worker Sizing                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Reglas generales:                                             │
│                                                                 │
│  • Workers = número de cores disponibles                       │
│  • CPU-bound workloads: workers = cores                        │
│  • I/O-bound workloads: workers < cores (deja room)           │
│  • Containers: respetar CPU limits                             │
│                                                                 │
│  Ejemplo para 8-core machine:                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Workload          │ Recommended Workers                 │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  HTTP proxy        │ 8 (1 per core)                     │  │
│  │  TLS termination   │ 6-8 (CPU intensive)                │  │
│  │  TCP proxy         │ 4-6 (I/O bound)                    │  │
│  │  With heavy WASM   │ 4-6 (WASM CPU overhead)            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Event Loop Tuning

```cpp
// Envoy usa libevent por defecto
// Algunas opciones pueden tunearse via config

// En la práctica, defaults son buenos para mayoría de casos
// Solo tunear si profiling muestra necesidad
```

---

## 4. Connection Tuning

### 4.1 Connection Limits

```yaml
# Limits por listener
static_resources:
  listeners:
    - name: listener_0
      per_connection_buffer_limit_bytes: 1048576 # 1MB
      listener_filters_timeout: 5s
      # Max connections (via iptables/nftables mejor)
```

### 4.2 Timeouts

```yaml
clusters:
  - name: backend
    connect_timeout: 5s

    # Para HTTP
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        common_http_protocol_options:
          idle_timeout: 3600s # 1 hour
        explicit_http_config:
          http2_protocol_options:
            initial_stream_window_size: 1048576 # 1MB
            initial_connection_window_size: 1048576
```

### 4.3 Connection Pooling

```yaml
clusters:
  - name: backend
    # Circuit breaker también limita connections
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 1000
          max_pending_requests: 1000
          max_requests: 10000
          max_retries: 3
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  Connection Pool Sizing                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Factors to consider:                                          │
│                                                                 │
│  1. Expected RPS                                               │
│  2. Average request duration                                   │
│  3. HTTP version (HTTP/2 multiplexes)                         │
│  4. Upstream limits                                            │
│                                                                 │
│  Formula (rough):                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ HTTP/1.1:                                                │  │
│  │   connections = RPS * avg_latency_seconds               │  │
│  │                                                          │  │
│  │ HTTP/2:                                                  │  │
│  │   connections = RPS * avg_latency / max_concurrent_streams│  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Example:                                                      │
│  • 10,000 RPS                                                  │
│  • 50ms average latency                                        │
│  • HTTP/1.1: 10000 * 0.05 = 500 connections                   │
│  • HTTP/2 (100 streams): 10000 * 0.05 / 100 = 5 connections   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Buffer Tuning

### 5.1 Request/Response Buffers

```yaml
http_filters:
  - name: envoy.filters.http.buffer
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.http.buffer.v3.Buffer
      max_request_bytes: 10485760 # 10MB max request body
```

### 5.2 Per-Connection Buffers

```yaml
listeners:
  - name: listener_0
    per_connection_buffer_limit_bytes: 1048576 # 1MB

# También en cluster
clusters:
  - name: backend
    per_connection_buffer_limit_bytes: 1048576
```

### 5.3 Watermarks

```
┌─────────────────────────────────────────────────────────────────┐
│                      Buffer Watermarks                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  High Watermark:                                               │
│  ─────────────────                                              │
│  Cuando buffer alcanza este nivel:                             │
│  • Deja de leer del socket                                     │
│  • Aplica backpressure                                         │
│  • Evita OOM                                                   │
│                                                                 │
│  Low Watermark:                                                │
│  ────────────────                                               │
│  Cuando buffer baja a este nivel:                              │
│  • Resume lectura                                              │
│  • Típicamente 50% del high                                    │
│                                                                 │
│  Buffer Diagram:                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                          │  │
│  │  ████████████████████ ← High Watermark (stop reading)   │  │
│  │  ██████████████                                          │  │
│  │  ██████████ ← Low Watermark (resume reading)            │  │
│  │  ████████                                                │  │
│  │  ████ ← Current buffer level                            │  │
│  │  ──                                                      │  │
│  │  Empty                                                   │  │
│  │                                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. TLS Optimization

### 6.1 Session Resumption

```yaml
transport_socket:
  name: envoy.transport_sockets.tls
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
    common_tls_context:
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
        cipher_suites:
          - ECDHE-RSA-AES128-GCM-SHA256 # Fast
          - ECDHE-RSA-AES256-GCM-SHA384
    session_ticket_keys:
      keys:
        - filename: /etc/envoy/session_ticket.key
```

### 6.2 OCSP Stapling

```yaml
common_tls_context:
  tls_certificates:
    - certificate_chain:
        filename: /etc/certs/server.crt
      private_key:
        filename: /etc/certs/server.key
      ocsp_staple:
        filename: /etc/certs/server.ocsp
```

### 6.3 TLS Performance Tips

```
┌─────────────────────────────────────────────────────────────────┐
│                    TLS Performance Tips                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Use TLS 1.3                                                │
│     • Faster handshake (1-RTT vs 2-RTT)                        │
│     • 0-RTT resumption possible                                │
│                                                                 │
│  2. Enable Session Resumption                                  │
│     • Session tickets o session cache                          │
│     • Reduce full handshakes                                   │
│                                                                 │
│  3. Choose Fast Ciphers                                        │
│     • AES-GCM con hardware support                             │
│     • ECDHE sobre DHE                                          │
│     • Avoid CBC ciphers                                        │
│                                                                 │
│  4. Use ECDSA Certificates                                     │
│     • Faster than RSA                                          │
│     • Smaller key sizes                                        │
│                                                                 │
│  5. OCSP Stapling                                              │
│     • Avoid client-side OCSP lookup                            │
│     • Faster connection establishment                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Filter Chain Optimization

### 7.1 Filter Order

```yaml
http_filters:
  # Orden importa!
  # Filters que rechazan rápido primero
  - name: envoy.filters.http.ratelimit # Reject early
  - name: envoy.filters.http.jwt_authn # Reject early
  - name: envoy.filters.http.cors
  - name: envoy.filters.http.buffer # Si necesitas buffer
  - name: envoy.filters.http.router # Siempre último
```

### 7.2 Deshabilitar Filters No Usados

```yaml
# Solo incluir filters que necesitas
# Cada filter añade overhead

# Por ejemplo, si no usas gRPC transcoding,
# no incluyas ese filter
```

### 7.3 Avoid Unnecessary Buffering

```
┌─────────────────────────────────────────────────────────────────┐
│              Streaming vs Buffering                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STREAMING (preferred when possible):                          │
│  ─────────────────────────────────────                          │
│  • Bytes fluyen directamente                                   │
│  • Baja latencia                                               │
│  • Bajo uso de memoria                                         │
│                                                                 │
│  BUFFERING (when needed):                                      │
│  ─────────────────────────                                      │
│  • Necesario para:                                             │
│    - Content modification                                      │
│    - Request signing                                           │
│    - Compression                                               │
│  • Aumenta latencia                                            │
│  • Aumenta memoria                                             │
│                                                                 │
│  Config para forzar streaming:                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ route:                                                   │  │
│  │   cluster: backend                                       │  │
│  │   auto_host_rewrite: true                               │  │
│  │   # NO usar filters que requieran buffering             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. Monitoring Performance

### 8.1 Key Stats to Monitor

```bash
# Latency
curl localhost:15000/stats | grep histogram

# Connections
curl localhost:15000/stats | grep "downstream_cx_active\|upstream_cx_active"

# Memory
curl localhost:15000/memory

# Server info
curl localhost:15000/server_info
```

### 8.2 Prometheus Queries

```promql
# P99 latency
histogram_quantile(0.99,
  rate(envoy_http_downstream_rq_time_bucket[5m]))

# Requests per second
rate(envoy_http_downstream_rq_total[1m])

# Active connections
envoy_http_downstream_cx_active

# Error rate
rate(envoy_http_downstream_rq_xx{envoy_response_code_class="5"}[5m]) /
rate(envoy_http_downstream_rq_total[5m])

# Memory usage
envoy_server_memory_allocated
envoy_server_memory_heap_size
```

---

## 9. Autoevaluación

1. ¿Cuáles son las 4 categorías principales de métricas de performance?
2. ¿Cómo capturas un CPU profile en Envoy?
3. ¿Cuántos workers deberías configurar por defecto?
4. ¿Qué son los buffer watermarks?
5. ¿Por qué el orden de los filters importa para performance?

---

## 10. Referencias

| Recurso                                                                                      | Descripción               |
| -------------------------------------------------------------------------------------------- | ------------------------- |
| [Envoy Performance](https://www.envoyproxy.io/docs/envoy/latest/faq/performance/performance) | FAQ de performance        |
| [pprof Guide](https://github.com/google/pprof)                                               | Herramienta pprof         |
| `bazel/PPROF.md`                                                                             | Guía de profiling en repo |
| [Stats Architecture](https://blog.envoyproxy.io/envoy-stats-b65c7f363342)                    | Blog sobre stats          |

---

**Siguiente**: [../ztunnel/rust_async_patterns.md](../ztunnel/rust_async_patterns.md) - Rust Async Patterns
