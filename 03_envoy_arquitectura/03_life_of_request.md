# Vida de un Request en Envoy

---
**Módulo**: 3 - Arquitectura de Envoy
**Tema**: Life of a Request
**Tiempo estimado**: 4 horas
**Prerrequisitos**: [02_threading_model.md](02_threading_model.md)
---

## Objetivos de Aprendizaje

Al completar este documento:
- Podrás trazar el camino completo de un request
- Identificarás cada componente involucrado
- Entenderás cómo fluyen los datos
- Podrás debuggear problemas en cualquier punto

---

## 1. Visión General del Flujo

```
┌─────────────────────────────────────────────────────────────────┐
│                    Life of a Request                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client                                                         │
│    │                                                            │
│    │ 1. TCP Connect                                             │
│    ▼                                                            │
│  ┌─────────────────┐                                           │
│  │    Listener     │ ─── 2. Accept connection                   │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 3. Create connection                                │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │ Network Filters │ ─── 4. L4 processing (TLS, etc.)          │
│  │   (L4 chain)    │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 5. If HTTP, pass to HCM                            │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │ HTTP Connection │ ─── 6. Parse HTTP, create stream          │
│  │    Manager      │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 7. Execute HTTP filter chain                        │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │  HTTP Filters   │ ─── 8. Auth, rate limit, etc.             │
│  │   (L7 chain)    │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 9. Router selects cluster                           │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │     Router      │ ─── 10. Match route config                │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 11. Get connection from pool                        │
│           ▼                                                     │
│  ┌─────────────────┐                                           │
│  │ Cluster Manager │ ─── 12. Load balance, health check        │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ 13. Forward request                                 │
│           ▼                                                     │
│       Upstream                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Paso 1-3: Connection Acceptance

### 2.1 Listener acepta conexión

```cpp
// source/common/listener_manager/listener_manager_impl.cc:200-400

// El listener escucha en un socket
void ListenerImpl::onAccept(Network::ConnectionSocketPtr&& socket) {
    // Seleccionar filter chain basado en:
    // - Destination IP/port
    // - Server name (SNI)
    // - Application protocol (ALPN)
    auto filter_chain = filter_chain_manager_.findFilterChain(*socket);

    // Crear conexión
    auto connection = dispatcher_.createConnection(
        std::move(socket),
        filter_chain.transportSocket(),
        filter_chain.networkFilterFactories()
    );

    // Añadir a la lista de conexiones activas
    connections_.push_back(std::move(connection));
}
```

### 2.2 Creación de Connection

```cpp
// source/common/network/connection_impl.h:50-150

class ConnectionImpl {
    // Socket file descriptor
    os_fd_t fd_;

    // Event dispatcher del worker
    Event::Dispatcher& dispatcher_;

    // Transport socket (raw o TLS)
    TransportSocketPtr transport_socket_;

    // Filter manager para procesar datos
    FilterManagerImpl filter_manager_;

    // Buffers
    Buffer::OwnedImpl read_buffer_;
    Buffer::OwnedImpl write_buffer_;
};
```

---

## 3. Paso 4: Network Filter Chain (L4)

### 3.1 Estructura de la Filter Chain

```
┌─────────────────────────────────────────────────────────────────┐
│                   Network Filter Chain                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Downstream bytes                                               │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────┐                                          │
│  │ TLS Inspector    │  ← Peek SNI without terminating TLS      │
│  └────────┬─────────┘                                          │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │ TLS Termination  │  ← Decrypt TLS (si está configurado)     │
│  └────────┬─────────┘                                          │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐                                          │
│  │ HTTP Connection  │  ← Si es HTTP, aquí empieza L7           │
│  │ Manager          │                                          │
│  └──────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Network Filter Interface

```cpp
// envoy/network/filter.h

class ReadFilter {
public:
    // Llamado cuando hay datos disponibles
    virtual FilterStatus onData(Buffer::Instance& data, bool end_stream) = 0;

    // Llamado cuando se establece la conexión
    virtual FilterStatus onNewConnection() = 0;
};

enum class FilterStatus {
    Continue,        // Pasar al siguiente filter
    StopIteration    // Pausar procesamiento
};
```

### 3.3 Código de TLS Termination

```cpp
// source/extensions/transport_sockets/tls/ssl_socket.cc:300-500

// OpenSSL handshake
PostIoAction SslSocket::doHandshake() {
    int rc = SSL_do_handshake(ssl_.get());
    if (rc == 1) {
        // Handshake completado
        handshake_complete_ = true;
        return PostIoAction::KeepOpen;
    }
    // ... manejo de errores y retry
}
```

---

## 4. Paso 5-6: HTTP Connection Manager

### 4.1 El Puente L4 → L7

El `HttpConnectionManager` es un **network filter** que entiende HTTP:

```cpp
// source/common/http/conn_manager_impl.h:60

class ConnectionManagerImpl : public Network::ReadFilter,
                              public Http::ConnectionCallbacks {
    // Codec para parsear HTTP
    ServerConnectionPtr codec_;

    // Lista de streams activos (requests)
    std::list<ActiveStreamPtr> streams_;

    // Configuración de rutas
    Router::ConfigConstSharedPtr route_config_;
};
```

### 4.2 Parsing HTTP

```cpp
// source/common/http/conn_manager_impl.cc:200-300

// Cuando llegan bytes
Network::FilterStatus ConnectionManagerImpl::onData(Buffer::Instance& data, bool) {
    // El codec parsea los bytes
    codec_->dispatch(data);
    // Los callbacks del codec crean streams
    return Network::FilterStatus::Continue;
}

// Callback cuando se parsean headers
void ConnectionManagerImpl::onHeadersComplete(StreamDecoderPtr&& decoder) {
    // Crear nuevo ActiveStream para este request
    auto stream = std::make_unique<ActiveStream>(*this, decoder);

    // Empezar a procesar
    stream->decodeHeaders(std::move(headers), end_stream);
}
```

### 4.3 Creación de Streams

```
HTTP/1.1: 1 request por conexión (o secuencial con keep-alive)
┌──────────────────────────────────────────┐
│ Connection                               │
│ ┌──────────────────────────────────────┐ │
│ │ Stream 1 (request 1)                 │ │
│ └──────────────────────────────────────┘ │
│ ┌──────────────────────────────────────┐ │
│ │ Stream 2 (request 2, después de 1)   │ │
│ └──────────────────────────────────────┘ │
└──────────────────────────────────────────┘

HTTP/2: Múltiples streams simultáneos
┌──────────────────────────────────────────┐
│ Connection                               │
│ ┌────────────────┐ ┌────────────────┐   │
│ │ Stream 1       │ │ Stream 3       │   │
│ └────────────────┘ └────────────────┘   │
│ ┌────────────────┐ ┌────────────────┐   │
│ │ Stream 5       │ │ Stream 7       │   │
│ └────────────────┘ └────────────────┘   │
└──────────────────────────────────────────┘
```

---

## 5. Paso 7-8: HTTP Filter Chain

### 5.1 Estructura de HTTP Filters

```cpp
// source/common/http/filter_manager.cc:100-300

class FilterManager {
    // Lista de decoder filters (request path)
    std::list<DecoderFilterPtr> decoder_filters_;

    // Lista de encoder filters (response path)
    std::list<EncoderFilterPtr> encoder_filters_;
};

// Interfaces
class StreamDecoderFilter {
    virtual FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers,
                                              bool end_stream) = 0;
    virtual FilterDataStatus decodeData(Buffer::Instance& data,
                                        bool end_stream) = 0;
    virtual FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) = 0;
};

class StreamEncoderFilter {
    virtual FilterHeadersStatus encodeHeaders(ResponseHeaderMap& headers,
                                              bool end_stream) = 0;
    virtual FilterDataStatus encodeData(Buffer::Instance& data,
                                        bool end_stream) = 0;
};
```

### 5.2 Ejemplo: JWT Filter

```cpp
// source/extensions/filters/http/jwt_authn/filter.cc:50-150

FilterHeadersStatus Filter::decodeHeaders(RequestHeaderMap& headers, bool) {
    // Extraer token del header Authorization
    auto auth_header = headers.get(Http::Headers::get().Authorization);
    if (!auth_header) {
        // Sin token
        return FilterHeadersStatus::Continue;  // O rechazar
    }

    // Validar JWT
    auto result = verifier_.verify(token);
    if (!result.ok()) {
        // Token inválido - enviar 401
        decoder_callbacks_->sendLocalReply(
            Http::Code::Unauthorized,
            "Invalid JWT",
            nullptr,
            absl::nullopt,
            "jwt_authn_failed"
        );
        return FilterHeadersStatus::StopIteration;
    }

    // Token válido - continuar
    return FilterHeadersStatus::Continue;
}
```

### 5.3 Flujo de Decode (Request)

```
Headers llegan
      │
      ▼
┌──────────────┐
│ Filter 1     │── decodeHeaders() ─────┐
│ (RBAC)       │                        │
└──────────────┘                        │
      │ Continue                        │
      ▼                                 │
┌──────────────┐                        │
│ Filter 2     │── decodeHeaders()      │ Si StopIteration,
│ (JWT)        │                        │ se detiene aquí
└──────────────┘                        │
      │ Continue                        │
      ▼                                 │
┌──────────────┐                        │
│ Filter 3     │── decodeHeaders()      │
│ (Rate Limit) │                        │
└──────────────┘                        │
      │ Continue                        │
      ▼                                 │
┌──────────────┐                        │
│ Router       │── Último filter        │
│              │   Forward to upstream  │
└──────────────┘                        │
```

---

## 6. Paso 9-10: Routing

### 6.1 Route Matching

```cpp
// source/common/router/config_impl.cc:500-800

RouteConstSharedPtr VirtualHostImpl::getRouteFromEntries(
    const Http::RequestHeaderMap& headers) {

    for (const auto& route : routes_) {
        // Match por path
        if (route->match().pathMatch(headers.Path())) {
            // Match por headers adicionales
            if (route->match().headersMatch(headers)) {
                return route;
            }
        }
    }
    return nullptr;
}
```

### 6.2 Configuración de Rutas

```yaml
route_config:
  virtual_hosts:
  - name: api
    domains: ["api.example.com"]
    routes:
    - match:
        prefix: "/api/v1/"
        headers:
        - name: "x-api-version"
          exact_match: "1"
      route:
        cluster: api-v1
        timeout: 30s
        retry_policy:
          retry_on: "5xx"
          num_retries: 3

    - match:
        prefix: "/api/v2/"
      route:
        cluster: api-v2
```

---

## 7. Paso 11-12: Cluster Manager

### 7.1 Selección de Cluster

```cpp
// source/common/upstream/cluster_manager_impl.cc:2046-2100

ClusterManager::getThreadLocalCluster(const std::string& cluster_name) {
    // Buscar en TLS (Thread Local Storage)
    auto it = thread_local_clusters_.find(cluster_name);
    if (it != thread_local_clusters_.end()) {
        return it->second;
    }
    return nullptr;
}
```

### 7.2 Load Balancing

```cpp
// source/common/upstream/load_balancer_impl.cc:200-400

HostConstSharedPtr RoundRobinLoadBalancer::chooseHost(LoadBalancerContext*) {
    // Obtener lista de hosts healthy
    const auto& hosts = priority_set_.hostSetsPerPriority()[0]->healthyHosts();

    if (hosts.empty()) {
        return nullptr;
    }

    // Round robin
    uint64_t index = rr_index_++ % hosts.size();
    return hosts[index];
}
```

### 7.3 Algoritmos de Load Balancing

| Algoritmo | Descripción | Uso |
|-----------|-------------|-----|
| `ROUND_ROBIN` | Rotación circular | Default, distribución uniforme |
| `LEAST_REQUEST` | Menos requests activos | Workloads variables |
| `RANDOM` | Aleatorio | Simple, bajo overhead |
| `RING_HASH` | Consistent hashing | Sticky sessions, caching |
| `MAGLEV` | Google's Maglev | Mejor distribución que ring hash |

---

## 8. Paso 13: Upstream Request

### 8.1 Connection Pool

```cpp
// source/common/http/conn_pool_base.cc:100-300

// Obtener conexión del pool
ConnectionPool::Cancellable* ConnPoolImpl::newStream(
    Http::ResponseDecoder& response_decoder,
    ConnectionPool::Callbacks& callbacks) {

    // ¿Hay conexión disponible?
    if (!ready_clients_.empty()) {
        // Usar conexión existente
        auto client = std::move(ready_clients_.front());
        ready_clients_.pop_front();
        return attachToClient(client, response_decoder, callbacks);
    }

    // ¿Podemos crear nueva?
    if (active_clients_.size() < max_connections_) {
        createNewConnection();
    }

    // Encolar request
    pending_requests_.push_back({&response_decoder, &callbacks});
    return pending_requests_.back().handle;
}
```

### 8.2 Forward Request

```cpp
// source/extensions/filters/http/router/router.cc:800-1000

void RouterFilter::onUpstreamHeaders(ResponseHeaderMapPtr&& headers, bool end_stream) {
    // Headers recibidos de upstream

    // Ejecutar encode filters (en reversa)
    decoder_callbacks_->encodeHeaders(std::move(headers), end_stream);
}
```

---

## 9. Response Path (Encode)

### 9.1 Flujo Inverso

```
Upstream response
      │
      ▼
┌──────────────┐
│ Router       │── encodeHeaders() ─────┐
│              │                        │
└──────────────┘                        │
      │                                 │
      ▼                                 │
┌──────────────┐                        │
│ Filter 3     │── encodeHeaders()      │
│ (Rate Limit) │   (puede añadir headers)
└──────────────┘                        │
      │                                 │
      ▼                                 │
┌──────────────┐                        │
│ Filter 2     │── encodeHeaders()      │
│ (JWT)        │                        │
└──────────────┘                        │
      │                                 │
      ▼                                 │
┌──────────────┐                        │
│ Filter 1     │── encodeHeaders()      │
│ (RBAC)       │                        │
└──────────────┘                        │
      │                                 │
      ▼                                 │
  HTTP Codec → TCP → Client
```

---

## 10. Logging y Metrics

### 10.1 Access Log

```cpp
// source/common/access_log/access_log_impl.cc:100-300

void AccessLogImpl::log(const Http::RequestHeaderMap* request_headers,
                        const Http::ResponseHeaderMap* response_headers,
                        const StreamInfo::StreamInfo& stream_info) {
    // Formatear según configuración
    std::string log_line = formatter_->format(
        request_headers, response_headers, stream_info
    );

    // Escribir a destino (file, gRPC, etc.)
    file_->write(log_line);
}
```

### 10.2 Stats Update

```cpp
// Después de completar request
void ActiveStream::onComplete() {
    // Incrementar contadores
    stats_.downstream_rq_total_.inc();

    if (response_code >= 200 && response_code < 300) {
        stats_.downstream_rq_2xx_.inc();
    } else if (response_code >= 500) {
        stats_.downstream_rq_5xx_.inc();
    }

    // Histograma de latencia
    stats_.downstream_rq_time_.recordValue(duration_ms);
}
```

---

## 11. Diagrama de Secuencia Completo

```
Client          Listener    HCM       Filter1   Filter2   Router    ClusterMgr  Upstream
  │                │          │          │         │         │          │          │
  │── TCP SYN ────>│          │          │         │         │          │          │
  │<── SYN-ACK ────│          │          │         │         │          │          │
  │── ACK ────────>│          │          │         │         │          │          │
  │                │          │          │         │         │          │          │
  │── HTTP Req ───>│          │          │         │         │          │          │
  │                │─ onData─>│          │         │         │          │          │
  │                │          │─decode──>│         │         │          │          │
  │                │          │          │─decode─>│         │          │          │
  │                │          │          │         │─decode─>│          │          │
  │                │          │          │         │         │─getCluster─>│       │
  │                │          │          │         │         │          │─connect─>│
  │                │          │          │         │         │          │          │
  │                │          │          │         │         │<───── Response ─────│
  │                │          │          │         │<─encode─│          │          │
  │                │          │          │<─encode─│         │          │          │
  │                │          │<─encode──│         │         │          │          │
  │<── HTTP Resp ──│          │          │         │         │          │          │
  │                │          │          │         │         │          │          │
```

---

## 12. Autoevaluación

1. ¿Cuál es el primer componente que toca un request TCP?
2. ¿Qué hace el HTTP Connection Manager?
3. ¿En qué orden se ejecutan los filters en el response path?
4. ¿Qué información usa el route matcher para seleccionar ruta?
5. ¿Por qué se usa connection pooling para upstream?

---

## 13. Ejercicio: Tracing un Request

Añade logging a puntos clave:

```cpp
// En conn_manager_impl.cc
ENVOY_LOG(info, "HCM: received request path={}", headers.Path());

// En filter
ENVOY_LOG(info, "Filter: processing headers");

// En router
ENVOY_LOG(info, "Router: selected cluster={}", cluster_name);
```

Ejecuta y observa el flujo en los logs.

---

**Siguiente**: [04_filter_chains.md](04_filter_chains.md) - Filter Chains en Detalle
