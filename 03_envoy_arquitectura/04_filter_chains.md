# Filter Chains en Envoy

---
**Módulo**: 3 - Arquitectura de Envoy
**Tema**: Network y HTTP Filter Chains
**Tiempo estimado**: 3 horas
**Prerrequisitos**: [03_life_of_request.md](03_life_of_request.md)
---

## Objetivos de Aprendizaje

Al completar este documento:
- Diferenciarás network filters de HTTP filters
- Entenderás cómo se encadenan y ejecutan
- Conocerás los filters más importantes
- Podrás configurar filter chains personalizadas

---

## 1. Tipos de Filters

### 1.1 Jerarquía de Filters

```
┌─────────────────────────────────────────────────────────────────┐
│                     Filter Hierarchy                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Listener Filters                                               │
│  ├── TLS Inspector (peek SNI)                                   │
│  └── HTTP Inspector (detect HTTP)                               │
│          │                                                      │
│          ▼                                                      │
│  Network Filters (L4)                                           │
│  ├── TCP Proxy                                                  │
│  ├── HTTP Connection Manager ──────────┐                        │
│  ├── Mongo Proxy                       │                        │
│  └── Redis Proxy                       │                        │
│                                        │                        │
│                                        ▼                        │
│                               HTTP Filters (L7)                 │
│                               ├── RBAC                          │
│                               ├── JWT Authentication            │
│                               ├── Rate Limit                    │
│                               ├── CORS                          │
│                               └── Router (terminal)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Resumen

| Tipo | Capa | Cuándo se ejecuta | Ejemplos |
|------|------|-------------------|----------|
| **Listener Filter** | Pre-connection | Antes de seleccionar filter chain | TLS Inspector |
| **Network Filter** | L4 | Por conexión | TCP Proxy, HCM |
| **HTTP Filter** | L7 | Por request (dentro de HCM) | Router, JWT, RBAC |

---

## 2. Network Filters

### 2.1 Ubicación en el Código

```
source/extensions/filters/network/
├── tcp_proxy/           # L4 proxy genérico
├── http_connection_manager/  # Puente a L7
├── mongo_proxy/         # Protocolo MongoDB
├── redis_proxy/         # Protocolo Redis
├── mysql_proxy/         # Protocolo MySQL
├── ext_authz/           # Autorización externa
└── ratelimit/           # Rate limiting L4
```

### 2.2 Interfaz de Network Filter

```cpp
// envoy/network/filter.h

class ReadFilter {
public:
    // Llamado cuando hay datos para leer
    virtual FilterStatus onData(Buffer::Instance& data,
                                bool end_stream) PURE;

    // Llamado cuando se establece conexión
    virtual FilterStatus onNewConnection() PURE;

    // Callbacks para interactuar con la conexión
    void setReadFilterCallbacks(ReadFilterCallbacks& callbacks);
};

class WriteFilter {
public:
    // Llamado cuando se escriben datos
    virtual FilterStatus onWrite(Buffer::Instance& data,
                                 bool end_stream) PURE;
};

// Un filter puede implementar ambos
class Filter : public ReadFilter, public WriteFilter { };
```

### 2.3 Ejemplo: TCP Proxy Filter

```cpp
// source/extensions/filters/network/tcp_proxy/tcp_proxy.cc

class TcpProxyFilter : public Network::ReadFilter {
public:
    FilterStatus onNewConnection() override {
        // Seleccionar upstream cluster
        auto cluster = cluster_manager_.getCluster(cluster_name_);

        // Conectar a upstream
        upstream_connection_ = cluster->tcpConnPool()->newConnection(*this);

        return FilterStatus::Continue;
    }

    FilterStatus onData(Buffer::Instance& data, bool end_stream) override {
        // Copiar datos a upstream
        upstream_connection_->write(data, end_stream);
        return FilterStatus::Continue;
    }
};
```

### 2.4 Configuración de Network Filter Chain

```yaml
listeners:
- name: tcp_listener
  filter_chains:
  - filters:
    # Primero: external authorization
    - name: envoy.filters.network.ext_authz
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.ext_authz.v3.ExtAuthz
        stat_prefix: ext_authz
        grpc_service:
          envoy_grpc:
            cluster_name: authz_cluster

    # Segundo: TCP proxy (terminal)
    - name: envoy.filters.network.tcp_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
        stat_prefix: tcp
        cluster: backend_cluster
```

---

## 3. HTTP Filters

### 3.1 Ubicación en el Código

```
source/extensions/filters/http/
├── router/              # Routing (siempre último)
├── rbac/                # Role-Based Access Control
├── jwt_authn/           # JWT authentication
├── ext_authz/           # External authorization
├── ratelimit/           # Rate limiting
├── local_ratelimit/     # Local rate limiting
├── cors/                # CORS handling
├── fault/               # Fault injection
├── buffer/              # Request buffering
├── compressor/          # Response compression
├── header_to_metadata/  # Header → metadata
├── lua/                 # Lua scripting
└── wasm/                # WebAssembly filters
```

### 3.2 Interfaz de HTTP Filter

```cpp
// envoy/http/filter.h

class StreamDecoderFilter {
public:
    // Request headers llegaron
    virtual FilterHeadersStatus decodeHeaders(
        RequestHeaderMap& headers,
        bool end_stream) PURE;

    // Request body data
    virtual FilterDataStatus decodeData(
        Buffer::Instance& data,
        bool end_stream) PURE;

    // Request trailers (HTTP/2)
    virtual FilterTrailersStatus decodeTrailers(
        RequestTrailerMap& trailers) PURE;
};

class StreamEncoderFilter {
public:
    // Response headers
    virtual FilterHeadersStatus encodeHeaders(
        ResponseHeaderMap& headers,
        bool end_stream) PURE;

    // Response body data
    virtual FilterDataStatus encodeData(
        Buffer::Instance& data,
        bool end_stream) PURE;
};
```

### 3.3 Return Values

```cpp
enum class FilterHeadersStatus {
    Continue,                    // Pasar al siguiente filter
    StopIteration,              // Pausar, esperar datos
    ContinueAndDontEndStream,   // Continuar pero puede haber más
    StopAllIterationAndBuffer,  // Pausar y buffear todo
    StopAllIterationAndWatermark // Pausar con watermarks
};

enum class FilterDataStatus {
    Continue,
    StopIterationAndBuffer,
    StopIterationAndWatermark,
    StopIterationNoBuffer
};
```

---

## 4. Filters Importantes en Detalle

### 4.1 Router Filter

El Router es **siempre el último** y es **terminal**:

```cpp
// source/extensions/filters/http/router/router.cc

FilterHeadersStatus RouterFilter::decodeHeaders(
    RequestHeaderMap& headers, bool end_stream) {

    // 1. Obtener ruta matching
    route_ = callbacks_->route();
    if (!route_) {
        callbacks_->sendLocalReply(Http::Code::NotFound, ...);
        return FilterHeadersStatus::StopIteration;
    }

    // 2. Obtener cluster
    cluster_ = cluster_manager_.getThreadLocalCluster(cluster_name);

    // 3. Crear upstream request
    upstream_request_ = std::make_unique<UpstreamRequest>(*this);

    // 4. Obtener conexión del pool
    conn_pool_->newStream(response_decoder, *this);

    return FilterHeadersStatus::StopIteration;
}
```

### 4.2 RBAC Filter

```cpp
// source/extensions/filters/http/rbac/rbac_filter.cc

FilterHeadersStatus RBACFilter::decodeHeaders(
    RequestHeaderMap& headers, bool) {

    // Evaluar policies
    auto result = engine_->handleAction(
        callbacks_->connection(),
        headers,
        callbacks_->streamInfo()
    );

    if (result == RBAC::Engine::Result::Deny) {
        callbacks_->sendLocalReply(
            Http::Code::Forbidden,
            "RBAC: access denied",
            nullptr, absl::nullopt, "rbac_access_denied"
        );
        return FilterHeadersStatus::StopIteration;
    }

    return FilterHeadersStatus::Continue;
}
```

**Configuración**:
```yaml
http_filters:
- name: envoy.filters.http.rbac
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
    rules:
      action: ALLOW
      policies:
        "service-admin":
          permissions:
          - any: true
          principals:
          - authenticated:
              principal_name:
                exact: "spiffe://cluster.local/ns/admin/sa/admin"
```

### 4.3 Rate Limit Filter

```yaml
http_filters:
- name: envoy.filters.http.local_ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
    stat_prefix: http_local_rate_limiter
    token_bucket:
      max_tokens: 1000
      tokens_per_fill: 100
      fill_interval: 1s
    filter_enabled:
      runtime_key: local_rate_limit_enabled
      default_value:
        numerator: 100
        denominator: HUNDRED
    filter_enforced:
      runtime_key: local_rate_limit_enforced
      default_value:
        numerator: 100
        denominator: HUNDRED
    response_headers_to_add:
    - append: false
      header:
        key: x-rate-limit-remaining
        value: "%REMAINING_TOKENS%"
```

### 4.4 Fault Injection Filter

Útil para testing de resiliencia:

```yaml
http_filters:
- name: envoy.filters.http.fault
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.fault.v3.HTTPFault
    delay:
      fixed_delay: 5s
      percentage:
        numerator: 10
        denominator: HUNDRED
    abort:
      http_status: 503
      percentage:
        numerator: 5
        denominator: HUNDRED
```

---

## 5. Filter Chain Execution

### 5.1 Decode Path (Request)

```
Request llega
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│ FilterManager::decodeHeaders()                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  for (filter : decoder_filters_) {                             │
│      status = filter->decodeHeaders(headers, end_stream);      │
│                                                                 │
│      if (status == StopIteration) {                            │
│          // Pausar, filter hará callback cuando esté listo     │
│          break;                                                 │
│      }                                                          │
│      // Continue → siguiente filter                             │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Encode Path (Response)

```
Response llega de upstream
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│ FilterManager::encodeHeaders()                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  // Orden INVERSO al decode                                     │
│  for (filter : reverse(encoder_filters_)) {                    │
│      status = filter->encodeHeaders(headers, end_stream);      │
│                                                                 │
│      if (status == StopIteration) {                            │
│          break;                                                 │
│      }                                                          │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Ejemplo Visual

```
Decode (Request):
  Filter1 → Filter2 → Filter3 → Router → Upstream

Encode (Response):
  Upstream → Router → Filter3 → Filter2 → Filter1 → Client

Si Filter2 quiere añadir header a response:
  - Se ejecuta en encodeHeaders()
  - Antes de que llegue a Filter1 y Client
```

---

## 6. Escribiendo un Custom Filter

### 6.1 Estructura Básica

```cpp
// my_filter.h
class MyFilter : public Http::StreamDecoderFilter {
public:
    // Constructor
    MyFilter(const MyFilterConfig& config);

    // StreamDecoderFilter interface
    FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers,
                                      bool end_stream) override;
    FilterDataStatus decodeData(Buffer::Instance& data,
                                bool end_stream) override;
    FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) override;

    void setDecoderFilterCallbacks(
        StreamDecoderFilterCallbacks& callbacks) override {
        decoder_callbacks_ = &callbacks;
    }

private:
    StreamDecoderFilterCallbacks* decoder_callbacks_{};
    const MyFilterConfig& config_;
};
```

### 6.2 Implementación

```cpp
// my_filter.cc
FilterHeadersStatus MyFilter::decodeHeaders(
    RequestHeaderMap& headers, bool end_stream) {

    // Leer header
    auto custom_header = headers.get(LowerCaseString("x-custom"));

    if (custom_header.empty()) {
        // Rechazar si falta header requerido
        decoder_callbacks_->sendLocalReply(
            Http::Code::BadRequest,
            "Missing x-custom header",
            nullptr, absl::nullopt, "missing_header"
        );
        return FilterHeadersStatus::StopIteration;
    }

    // Añadir metadata para downstream filters
    decoder_callbacks_->streamInfo().filterState()->setData(
        "my_filter.custom_value",
        std::make_unique<StringAccessorImpl>(custom_header[0]->value()),
        StreamInfo::FilterState::StateType::ReadOnly
    );

    return FilterHeadersStatus::Continue;
}
```

### 6.3 Factory

```cpp
// my_filter_config.cc
class MyFilterFactory : public NamedHttpFilterConfigFactory {
public:
    Http::FilterFactoryCb createFilterFactoryFromProto(
        const Protobuf::Message& proto_config,
        const std::string&,
        FactoryContext& context) override {

        auto config = MessageUtil::downcastAndValidate<MyFilterProto>(
            proto_config);

        return [config](Http::FilterChainFactoryCallbacks& callbacks) {
            callbacks.addStreamDecoderFilter(
                std::make_shared<MyFilter>(config));
        };
    }

    std::string name() const override { return "my_filter"; }
};

REGISTER_FACTORY(MyFilterFactory,
                 Server::Configuration::NamedHttpFilterConfigFactory);
```

---

## 7. Configuración Completa de HTTP Filters

```yaml
http_filters:
# 1. CORS (primero para preflight)
- name: envoy.filters.http.cors
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors

# 2. Rate limiting
- name: envoy.filters.http.local_ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
    # ...config...

# 3. JWT Authentication
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
    # ...config...

# 4. RBAC Authorization
- name: envoy.filters.http.rbac
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
    # ...config...

# 5. Router (SIEMPRE último)
- name: envoy.filters.http.router
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

---

## 8. Autoevaluación

1. ¿Cuál es la diferencia entre network filter y HTTP filter?
2. ¿Por qué el Router debe ser el último HTTP filter?
3. ¿Qué significa `FilterHeadersStatus::StopIteration`?
4. ¿En qué orden se ejecutan los filters en el response path?
5. ¿Cómo un filter puede rechazar un request?

---

**Siguiente**: [05_cluster_management.md](05_cluster_management.md) - Cluster Management
