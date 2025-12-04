# Referencias y Recursos

Este documento contiene enlaces a documentación oficial, artículos y recursos de aprendizaje.

---

## Documentación Oficial

### Envoy
- [Envoy Documentation](https://www.envoyproxy.io/docs/envoy/latest/)
- [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
- [Threading Model](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/threading_model)
- [API Reference](https://www.envoyproxy.io/docs/envoy/latest/api-v3/api)
- [GitHub Repository](https://github.com/envoyproxy/envoy)

### Istio
- [Istio Documentation](https://istio.io/latest/docs/)
- [Ambient Mode Overview](https://istio.io/latest/docs/ambient/overview/)
- [Ambient Architecture](https://istio.io/latest/docs/ambient/architecture/)
- [Traffic Redirection](https://istio.io/latest/docs/ambient/architecture/traffic-redirection/)

### ztunnel
- [ztunnel GitHub](https://github.com/istio/ztunnel)
- [ztunnel Architecture (Istio repo)](https://github.com/istio/istio/blob/master/architecture/ambient/ztunnel.md)
- [Rust-Based Ztunnel Blog](https://istio.io/latest/blog/2023/rust-based-ztunnel/)

---

## Artículos y Blogs

### Proxies L4 vs L7
- [Layer 4 vs Layer 7 Proxy Mode (HAProxy)](https://www.haproxy.com/blog/layer-4-and-layer-7-proxy-mode)
- [L4/L7 Proxy Tutorial (Tencent)](https://edgeone.ai/learning/l4-l7-proxy)
- [Layer 4 vs Layer 7 Load Balancing (HAProxy)](https://www.haproxy.com/blog/layer-4-vs-layer-7-load-balancing)

### Envoy Internals
- [Envoy Threading Model (Blog)](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)
- [Envoy Stats Architecture](https://medium.com/@mattklein123/envoy-stats-b65c7f363342)
- [Envoy Hot Restart](https://medium.com/@mattklein123/envoy-hot-restart-1d16b14555b5)

### Istio Ambient
- [Understanding Ztunnel (Solo.io)](https://www.solo.io/blog/understanding-istio-ambient-ztunnel-and-secure-overlay)
- [Istio Ambient with OpenShift (Red Hat)](https://next.redhat.com/2024/03/19/istio-ambient-mode-with-red-hat-openshift/)

---

## Especificaciones y RFCs

### Networking
- [OSI Model (Wikipedia)](https://en.wikipedia.org/wiki/OSI_model)
- [TCP/IP Model](https://en.wikipedia.org/wiki/Internet_protocol_suite)
- [RFC 793 - TCP](https://tools.ietf.org/html/rfc793)
- [RFC 768 - UDP](https://tools.ietf.org/html/rfc768)

### HTTP
- [RFC 7230-7235 - HTTP/1.1](https://tools.ietf.org/html/rfc7230)
- [RFC 7540 - HTTP/2](https://tools.ietf.org/html/rfc7540)
- [RFC 9114 - HTTP/3](https://tools.ietf.org/html/rfc9114)
- [gRPC Documentation](https://grpc.io/docs/)

### Security
- [RFC 8446 - TLS 1.3](https://tools.ietf.org/html/rfc8446)
- [SPIFFE Specification](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/)
- [RFC 7519 - JWT](https://tools.ietf.org/html/rfc7519)

---

## Repositorios Locales

```bash
# Envoy
/home/jojosneg/source/redhat/envoy/upstream/main

# ztunnel
/home/jojosneg/source/redhat/ztunnel/upstream/master
```

---

## Archivos Clave para Estudio

### Envoy

| Archivo | Descripción |
|---------|-------------|
| `docs/analysis/00_architecture_overview.md` | Arquitectura completa con diagramas |
| `source/common/http/conn_manager_impl.h` | HTTP Connection Manager |
| `source/extensions/filters/http/router/router.cc` | Router filter |
| `source/common/upstream/cluster_manager_impl.h` | Cluster management |
| `source/common/upstream/load_balancer_impl.cc` | Load balancing algorithms |
| `source/common/event/dispatcher_impl.h` | Event loop |
| `source/server/worker_impl.h` | Worker threads |

### ztunnel

| Archivo | Descripción |
|---------|-------------|
| `ARCHITECTURE.md` | Threading model y puertos |
| `README.md` | Overview, métricas, logging |
| `src/proxy/` | Core proxy implementation |
| `src/xds/` | xDS client |
| `src/identity/` | SPIFFE certificate management |

---

## Herramientas Útiles

### Networking
```bash
# Captura de tráfico
tcpdump -i any port 8080

# Ver conexiones
ss -tan

# DNS lookup
dig example.com

# HTTP testing
curl -v http://localhost:8080
```

### Envoy
```bash
# Admin interface
curl localhost:9901/stats
curl localhost:9901/config_dump
curl localhost:9901/clusters

# Compilar
bazel build -c opt //source/exe:envoy-static

# Tests
bazel test //test/common/http:async_client_impl_test
```

### ztunnel
```bash
# Compilar
cargo build

# Tests
cargo test

# Ver versión Rust requerida
BUILD_WITH_CONTAINER=1 make rust-version
```

### gRPC
```bash
# Listar servicios
grpcurl -plaintext localhost:50051 list

# Describir servicio
grpcurl -plaintext localhost:50051 describe MyService

# Llamar método
grpcurl -plaintext -d '{"id": "123"}' localhost:50051 MyService/GetItem
```

---

## Conferencias y Charlas

- EnvoyCon (parte de KubeCon)
- IstioCon
- KubeCon + CloudNativeCon

---

*Última actualización: 2025-12-04*
