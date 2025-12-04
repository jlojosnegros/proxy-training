# Proxy Layer 4 en Profundidad

---

**Módulo**: 2 - Proxies - Conceptos Fundamentales
**Tema**: Proxy L4
**Tiempo estimado**: 2 horas
**Prerrequisitos**: [01_que_es_proxy.md](01_que_es_proxy.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás cómo opera un proxy L4
- Sabrás qué decisiones puede tomar y cuáles no
- Comprenderás el modelo de ztunnel como proxy L4
- Identificarás casos de uso ideales para L4

---

## 1. Características de un Proxy L4

### 1.1 Qué "Ve" un Proxy L4

Un proxy L4 opera en la **capa de transporte** del modelo OSI. Solo tiene acceso a:

```
┌─────────────────────────────────────────────────────────────────┐
│ Lo que VE un proxy L4                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ IP Header                                                 │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Source IP:      10.0.1.5                           ✓     │  │
│  │ Destination IP: 10.0.2.10                          ✓     │  │
│  │ Protocol:       TCP (6)                            ✓     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TCP Header                                                │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Source Port:      45678                            ✓     │  │
│  │ Destination Port: 8080                             ✓     │  │
│  │ Flags:            SYN, ACK, etc.                   ✓     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Payload (Application Data)                                │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ GET /api/users HTTP/1.1                            ✗     │  │
│  │ Host: api.example.com                              ✗     │  │
│  │ Authorization: Bearer eyJ...                       ✗     │  │
│  │                                                    │     │  │
│  │ (El proxy L4 ve bytes opacos, no entiende HTTP)   │     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Qué Puede y No Puede Hacer

| Capacidad                         | Proxy L4 | Por qué                     |
| --------------------------------- | -------- | --------------------------- |
| **Routing por IP/puerto**         | ✓        | Visible en headers L3/L4    |
| **mTLS**                          | ✓        | TLS opera en L4/L5          |
| **Connection-level metrics**      | ✓        | Bytes, conexiones, latencia |
| **Load balancing**                | ✓        | Puede seleccionar backend   |
| **Routing por URL path**          | ✗        | Path está en payload HTTP   |
| **Routing por header**            | ✗        | Headers están en payload    |
| **JWT validation**                | ✗        | JWT está en payload         |
| **gRPC method routing**           | ✗        | Método está en HTTP headers |
| **Request/response modification** | ✗        | No entiende el protocolo    |

### 1.3 El Flujo de un Proxy L4

```
1. ACCEPT CONNECTION
   ┌────────────────────────────────────────────────────────────┐
   │ Downstream client conecta al proxy                         │
   │ Proxy acepta: source=10.0.1.5:45678, dest=proxy:8080      │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
2. SELECT UPSTREAM (basado en IP:puerto destino)
   ┌────────────────────────────────────────────────────────────┐
   │ Si dest=10.0.2.10:8080 → cluster "backend"                │
   │ Load balancer selecciona: 10.0.3.1:8080                   │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
3. ESTABLISH UPSTREAM CONNECTION
   ┌────────────────────────────────────────────────────────────┐
   │ Proxy conecta a 10.0.3.1:8080                             │
   │ (Opcionalmente con mTLS)                                   │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
4. BIDIRECTIONAL BYTE COPY
   ┌────────────────────────────────────────────────────────────┐
   │ downstream → upstream: copia bytes                         │
   │ upstream → downstream: copia bytes                         │
   │ (Sin interpretar contenido)                                │
   └────────────────────────────────────────────────────────────┘
                              │
                              ▼
5. CLOSE CONNECTIONS
   ┌────────────────────────────────────────────────────────────┐
   │ FIN de cualquier lado → cerrar ambas conexiones           │
   │ Emitir métricas: bytes_sent, bytes_recv, duration          │
   └────────────────────────────────────────────────────────────┘
```

---

## 2. ztunnel como Proxy L4

### 2.1 Arquitectura de ztunnel

ztunnel es un **proxy L4 puro** diseñado específicamente para Istio ambient mode:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Node                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      ztunnel                             │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │                                                         │   │
│  │  Port 15001 ─────────► Outbound traffic capture         │   │
│  │  Port 15006 ─────────► Inbound plaintext capture        │   │
│  │  Port 15008 ─────────► HBONE (mTLS tunnel)              │   │
│  │                                                         │   │
│  │  ┌───────────────────────────────────────────────────┐ │   │
│  │  │ Funciones:                                         │ │   │
│  │  │ ✓ mTLS entre pods                                  │ │   │
│  │  │ ✓ Identity (SPIFFE)                                │ │   │
│  │  │ ✓ L4 Authorization                                 │ │   │
│  │  │ ✓ Telemetry (conexiones, bytes)                    │ │   │
│  │  │ ✗ HTTP routing                                     │ │   │
│  │  │ ✗ HTTP header manipulation                         │ │   │
│  │  │ ✗ JWT validation                                   │ │   │
│  │  └───────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│            ┌──────────────┼──────────────┐                     │
│            │              │              │                     │
│            ▼              ▼              ▼                     │
│       ┌────────┐    ┌────────┐    ┌────────┐                  │
│       │ Pod A  │    │ Pod B  │    │ Pod C  │                  │
│       │ (app)  │    │ (app)  │    │ (app)  │                  │
│       └────────┘    └────────┘    └────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Flujo de Tráfico en ztunnel

**Outbound (Pod → External)**:

```
┌────────┐      iptables        ┌──────────┐       HBONE        ┌──────────┐
│ Pod A  │ ─────redirect───────>│ ztunnel  │─────mTLS────────>│ ztunnel  │
│        │      to 15001        │  (node)  │     port 15008    │ (remote) │
│app:8080│                      │          │                   │          │
└────────┘                      └──────────┘                   └──────────┘
     │                               │
     └── App cree que conecta        └── ztunnel:
         a 10.0.2.5:80                   1. Intercepta conexión
                                          2. Identifica destino
                                          3. Establece HBONE tunnel
                                          4. Copia bytes
```

**Inbound (External → Pod)**:

```
┌──────────┐     HBONE         ┌──────────┐     plaintext    ┌────────┐
│ ztunnel  │────mTLS──────────>│ ztunnel  │─────redirect────>│ Pod B  │
│ (remote) │    port 15008     │  (node)  │    to localhost  │        │
│          │                   │          │                  │app:8080│
└──────────┘                   └──────────┘                  └────────┘
                                    │
                                    └── ztunnel:
                                         1. Recibe HBONE
                                         2. Valida mTLS
                                         3. Desencapsula
                                         4. Envía a pod
```

### 2.3 Código de ztunnel

El core del proxy está en:

```
src/proxy/
```

**Estructura típica de un proxy L4 en Rust**:

```rust
// Pseudocódigo conceptual
async fn handle_connection(downstream: TcpStream) -> Result<()> {
    // 1. Determinar destino original
    let original_dest = get_original_dest(&downstream)?;

    // 2. Establecer conexión upstream con mTLS
    let upstream = if is_in_mesh(&original_dest) {
        establish_hbone_connection(original_dest).await?
    } else {
        TcpStream::connect(original_dest).await?
    };

    // 3. Copiar bytes bidireccionalmente
    let (dr, dw) = downstream.split();
    let (ur, uw) = upstream.split();

    tokio::select! {
        r = tokio::io::copy(dr, uw) => r?,
        r = tokio::io::copy(ur, dw) => r?,
    }

    // 4. Emitir métricas
    emit_metrics(bytes_sent, bytes_received, duration);

    Ok(())
}
```

---

## 3. TCP Proxy en Envoy

Envoy también puede actuar como proxy L4 usando el filtro `tcp_proxy`:

### 3.1 Configuración

```yaml
static_resources:
  listeners:
    - name: tcp_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 10000
      filter_chains:
        - filters:
            - name: envoy.filters.network.tcp_proxy
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
                stat_prefix: tcp_stats
                cluster: backend_cluster

  clusters:
    - name: backend_cluster
      connect_timeout: 5s
      type: STATIC
      load_assignment:
        cluster_name: backend_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 10.0.1.5
                      port_value: 8080
```

### 3.2 Código

```
source/extensions/filters/network/tcp_proxy/tcp_proxy.cc
```

Este filtro:

1. Recibe conexión downstream
2. Selecciona upstream basado en cluster config
3. Conecta a upstream
4. Copia bytes en ambas direcciones
5. Cierra cuando cualquier lado cierra

---

## 4. Casos de Uso Ideales para L4

### 4.1 mTLS Transparente

Cuando solo necesitas cifrado sin L7:

```
┌────────┐                    ┌────────┐
│ App A  │◄──── mTLS ────────►│ App B  │
│        │      (L4)          │        │
│ HTTP   │                    │ HTTP   │
│ gRPC   │  Las apps no saben │ gRPC   │
│ Redis  │  que hay mTLS      │ Redis  │
└────────┘                    └────────┘
```

### 4.2 Protocolos No-HTTP

Para protocolos que Envoy no parsea nativamente:

| Protocolo     | Proxy L4    | Proxy L7                  |
| ------------- | ----------- | ------------------------- |
| MySQL         | ✓ TCP proxy | Requiere codec específico |
| Redis         | ✓ TCP proxy | Envoy tiene redis_proxy   |
| Kafka         | ✓ TCP proxy | Requiere codec            |
| Custom binary | ✓ TCP proxy | No soportado              |

### 4.3 Alto Rendimiento

Cuando la latencia es crítica:

```
Comparación de overhead:

L4 Proxy:
  Parse: IP header + TCP header
  Time: ~microseconds

L7 Proxy:
  Parse: IP + TCP + HTTP headers + body (si aplica)
  Process: Filters, routing rules
  Time: ~milliseconds

Para 1 millón de req/s, la diferencia importa.
```

### 4.4 Service Mesh a Escala

```
Cluster con 1000 pods:

Modelo Sidecar (L7):
  1000 Envoy sidecars
  ~100MB RAM cada uno
  = ~100GB RAM total

Modelo Node Proxy (L4):
  50 ztunnels (50 nodos)
  ~50MB RAM cada uno
  = ~2.5GB RAM total
```

---

## 5. Limitaciones de L4

### 5.1 No Puede Hacer Routing Inteligente

```
Request 1: GET /api/v1/users
Request 2: GET /api/v2/users

Un proxy L4 ve ambos como:
  "TCP bytes to 10.0.1.5:8080"

No puede:
  - Enviar v1 a cluster-v1
  - Enviar v2 a cluster-v2
```

### 5.2 No Puede Validar Auth por Request

```
Request con JWT válido → ✓
Request con JWT inválido → ✗

Un proxy L4 ve ambos como bytes.
No puede rechazar el JWT inválido.
```

### 5.3 No Tiene Request-Level Metrics

```
L7 puede reportar:
  - Requests por segundo
  - Latencia por endpoint (/api/users vs /api/orders)
  - Errores 5xx por path

L4 solo puede reportar:
  - Bytes por segundo
  - Conexiones activas
  - Latencia de conexión
```

---

## 6. Autoevaluación

1. ¿Qué información puede ver un proxy L4 de un paquete?
2. ¿Por qué ztunnel puede hacer mTLS pero no routing por URL?
3. ¿Cuál es el overhead aproximado de un proxy L4 vs L7?
4. ¿Para qué tipo de protocolos es ideal un proxy L4?
5. ¿Qué métrica puede reportar L7 que L4 no puede?

---

## 7. Referencias en el Código

### ztunnel

| Archivo           | Descripción               |
| ----------------- | ------------------------- |
| `src/proxy/`      | Core proxy implementation |
| `ARCHITECTURE.md` | Puertos y flujo           |

### Envoy

| Archivo                                                    | Descripción         |
| ---------------------------------------------------------- | ------------------- |
| `source/extensions/filters/network/tcp_proxy/tcp_proxy.cc` | TCP proxy filter    |
| `source/common/network/connection_impl.h`                  | Connection handling |

---

**Siguiente**: [03_proxy_l7.md](03_proxy_l7.md) - Proxy Layer 7 en Profundidad
