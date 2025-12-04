# Ejercicio: Trazar un Request en Envoy

---

**Tipo**: Ejercicio práctico
**Tiempo estimado**: 1-2 horas
**Prerrequisitos**: Ejercicio 01 completado

---

## Objetivo

Entender el flujo de un request a través de Envoy usando logging, stats y herramientas de debugging.

---

## 1. Configurar Access Logging

### 1.1 Añadir Access Log a la Configuración

Crea `trace_config.yaml`:

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 15000

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
                codec_type: AUTO

                # Access logging
                access_log:
                  - name: envoy.access_loggers.stdout
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
                      log_format:
                        text_format: |
                          [%START_TIME%] "%REQ(:METHOD)% %REQ(:PATH)% %PROTOCOL%"
                          Response: %RESPONSE_CODE% %RESPONSE_FLAGS%
                          Duration: %DURATION%ms
                          Upstream: %UPSTREAM_HOST%
                          Request ID: %REQ(X-REQUEST-ID)%
                          ---

                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/api"
                          route:
                            cluster: api_cluster
                        - match:
                            prefix: "/"
                          route:
                            cluster: default_cluster

                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: api_cluster
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: api_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: httpbin.org
                      port_value: 80

    - name: default_cluster
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: default_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: httpbin.org
                      port_value: 80
```

### 1.2 Iniciar Envoy

```bash
./bazel-bin/source/exe/envoy-static -c trace_config.yaml
```

---

## 2. Hacer Requests y Observar Logs

### 2.1 Request Simple

```bash
# En otra terminal
curl http://localhost:8080/get
```

**Observar en logs de Envoy:**

```
[2025-12-04T10:30:00.000Z] "GET /get HTTP/1.1"
Response: 200 -
Duration: 150ms
Upstream: 54.175.219.8:80
Request ID: abc123-...
---
```

### 2.2 Request a Path Específico

```bash
curl http://localhost:8080/api/users
```

**Observar:** El request va al `api_cluster`

### 2.3 Request con Headers

```bash
curl -H "X-Custom-Header: test" http://localhost:8080/headers
```

---

## 3. Usar Debug Logging

### 3.1 Habilitar Debug via Admin

```bash
# Habilitar debug para http
curl -X POST "http://localhost:15000/logging?http=debug"

# Ahora hacer un request
curl http://localhost:8080/get
```

**Observar logs detallados:**

- Parsing de headers
- Routing decision
- Upstream selection

### 3.2 Trace Level

```bash
# Para aún más detalle
curl -X POST "http://localhost:15000/logging?http=trace"
```

---

## 4. Examinar Estadísticas

### 4.1 Stats Relevantes

```bash
# Ver todas las stats
curl http://localhost:15000/stats | grep -E "downstream_rq|upstream_rq"

# Stats de request
curl http://localhost:15000/stats | grep downstream_rq_total

# Stats de respuesta por código
curl http://localhost:15000/stats | grep downstream_rq_2xx

# Stats de upstream
curl http://localhost:15000/stats | grep upstream_rq_time
```

### 4.2 Stats por Cluster

```bash
# Stats del api_cluster
curl "http://localhost:15000/stats?filter=cluster.api_cluster"

# Stats del default_cluster
curl "http://localhost:15000/stats?filter=cluster.default_cluster"
```

### 4.3 Histogramas de Latencia

```bash
# Ver distribución de latencia
curl http://localhost:15000/stats | grep histogram

# Percentiles
curl http://localhost:15000/stats | grep "\.quantile"
```

---

## 5. Tracing con Request ID

### 5.1 Añadir Request ID

```bash
# Enviar request con ID explícito
curl -H "x-request-id: my-trace-123" http://localhost:8080/get
```

### 5.2 Buscar en Logs

Buscar en los logs por `my-trace-123` para ver todo el flujo.

---

## 6. Examinar Config Dump

### 6.1 Ver Configuración Actual

```bash
# Config completa
curl http://localhost:15000/config_dump | jq .

# Solo listeners
curl "http://localhost:15000/config_dump?resource=listeners" | jq .

# Solo clusters
curl "http://localhost:15000/config_dump?resource=clusters" | jq .

# Solo routes
curl "http://localhost:15000/config_dump?resource=routes" | jq .
```

### 6.2 Ver Route Matching

```bash
# Ver cómo se resuelve un path
curl "http://localhost:15000/config_dump?resource=routes" | jq '.configs[].dynamic_route_configs'
```

---

## 7. Ejercicio: Trazar Request Fallido

### 7.1 Configurar Cluster que Falle

Añadir a la configuración:

```yaml
clusters:
  # ... clusters anteriores ...
  - name: failing_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    connect_timeout: 1s
    load_assignment:
      cluster_name: failing_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: nonexistent.invalid
                    port_value: 80
```

Y una ruta:

```yaml
routes:
  - match:
      prefix: "/fail"
    route:
      cluster: failing_cluster
```

### 7.2 Observar Fallo

```bash
curl http://localhost:8080/fail
```

**Observar:**

- Response code: 503
- Response flags en access log
- Stats de error

```bash
# Ver stats de errores
curl http://localhost:15000/stats | grep -E "upstream_rq_5xx|upstream_cx_connect_fail"
```

---

## 8. Checkpoints de Verificación

- [ ] Access logging funcionando
- [ ] Debug logging habilitado vía admin
- [ ] Stats de downstream_rq visibles
- [ ] Stats de upstream_rq visibles
- [ ] Request ID propagándose
- [ ] Config dump accesible
- [ ] Errores correctamente loggeados

---

## 9. Variables de Access Log Útiles

| Variable             | Descripción                   |
| -------------------- | ----------------------------- |
| `%START_TIME%`       | Timestamp del request         |
| `%REQ(:METHOD)%`     | HTTP method                   |
| `%REQ(:PATH)%`       | Request path                  |
| `%PROTOCOL%`         | HTTP/1.1 o HTTP/2             |
| `%RESPONSE_CODE%`    | HTTP response code            |
| `%RESPONSE_FLAGS%`   | Flags de Envoy (UF, NR, etc.) |
| `%DURATION%`         | Duración total en ms          |
| `%UPSTREAM_HOST%`    | IP:port del upstream          |
| `%UPSTREAM_CLUSTER%` | Nombre del cluster            |

---

## 10. Response Flags Comunes

| Flag | Significado                       |
| ---- | --------------------------------- |
| `UF` | Upstream connection failure       |
| `NR` | No route configured               |
| `UC` | Upstream connection termination   |
| `UT` | Upstream request timeout          |
| `LH` | Local service failed health check |
| `DC` | Downstream connection termination |

---

**Siguiente ejercicio**: [03_modificar_filtro.md](03_modificar_filtro.md) - Modificar un Filtro
