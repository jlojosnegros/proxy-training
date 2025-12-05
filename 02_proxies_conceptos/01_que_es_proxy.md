# ¿Qué es un Proxy de Red?

---

**Módulo**: 2 - Proxies - Conceptos Fundamentales
**Tema**: Introducción a Proxies
**Tiempo estimado**: 2 horas
**Prerrequisitos**: Módulo 1 completo

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás qué es un proxy y por qué existen
- Diferenciarás entre forward y reverse proxy
- Comprenderás los casos de uso principales
- Identificarás dónde encajan Envoy y ztunnel

---

## 1. Definición de Proxy

### 1.1 Concepto Básico

Un **proxy** es un intermediario que actúa en nombre de otro. En redes, un proxy de red es un servidor que recibe requests de clientes y los reenvía a servidores destino:

```mermaid
flowchart LR
    subgraph NoProxy["Sin proxy"]
        C1["Client"] <-->|Request/Response| S1["Server"]
    end

    subgraph WithProxy["Con proxy"]
        C2["Client"] <--> P["Proxy"] <--> S2["Server"]
    end
```

**El proxy puede:**
- Inspeccionar tráfico
- Modificar requests/responses
- Cachear contenido
- Aplicar políticas
- Agregar observabilidad

### 1.2 ¿Por Qué Usar un Proxy?

| Objetivo               | Descripción                          | Ejemplo                      |
| ---------------------- | ------------------------------------ | ---------------------------- |
| **Seguridad**          | Cifrado, autenticación, autorización | mTLS, JWT validation         |
| **Observabilidad**     | Métricas, logs, traces               | Request latency, error rates |
| **Resiliencia**        | Retries, circuit breakers, timeouts  | Failover automático          |
| **Control de tráfico** | Rate limiting, load balancing        | Proteger backends            |
| **Transformación**     | Modificar headers, body, protocol    | gRPC-JSON transcoding        |
| **Caché**              | Almacenar responses frecuentes       | Reducir carga upstream       |

---

## 2. Tipos de Proxies

### 2.1 Forward Proxy (Proxy Directo)

El cliente conoce el proxy y lo configura explícitamente:

```mermaid
flowchart LR
    subgraph Forward["Forward Proxy"]
        C["Client<br/>config: proxy=:8080"] --> P["Proxy"]
        P --> I["internet.com<br/>api.external.com<br/>any server"]
    end
```

**Casos de uso:**
- Filtrado de contenido
- Control de acceso a internet
- Anonimizar origen
- Caché corporativo

**Ejemplos**: Squid, Proxy corporativo

### 2.2 Reverse Proxy (Proxy Inverso)

Los clientes no saben que hay un proxy; creen hablar directamente con el servidor:

```mermaid
flowchart LR
    subgraph Reverse["Reverse Proxy"]
        C["Client<br/>request to:<br/>api.example.com"] --> P["Proxy<br/>api.example.com"]
        P --> B1["Backend 1"]
        P --> B2["Backend 2"]
        P --> B3["Backend 3"]
    end
```

**Casos de uso:**
- Load balancing
- SSL termination
- API Gateway
- Proteger backends
- Service Mesh sidecar

**Ejemplos**: Nginx, HAProxy, Envoy, ztunnel

**Envoy y ztunnel son principalmente reverse proxies**, aunque Envoy puede actuar como forward proxy también.

### 2.3 Transparent Proxy (Proxy Transparente)

El tráfico se redirige al proxy sin que el cliente lo sepa o configure:

```mermaid
flowchart LR
    subgraph Transparent["Transparent Proxy"]
        C["Client<br/>to: 10.0.2.5:80"] -->|Request| IPT["iptables<br/>redirect"]
        IPT --> P["Proxy"]
        P --> S["Server<br/>10.0.2.5:80"]
        S --> P
        P --> C
    end
```

**Mecanismo:**
- iptables/netfilter
- TPROXY
- Network namespace manipulation

**Usado por:**
- ztunnel (Istio ambient)
- Istio sidecar (iptables redirect)

**ztunnel usa transparent proxying**:

```
ARCHITECTURE.md:
| 15001 | Pod outbound traffic capture  | Y |
| 15006 | Pod inbound plaintext capture | Y |
```

Las reglas iptables redirigen el tráfico de los pods al ztunnel sin que los pods lo sepan.

---

## 3. Proxy en el Contexto de Service Mesh

### 3.1 Sidecar Proxy (Modelo Tradicional)

Cada pod tiene su propio proxy sidecar:

```mermaid
flowchart LR
    subgraph PodA["Pod A"]
        AppA["App"] --> EA["Envoy Sidecar"]
    end

    subgraph PodB["Pod B"]
        EB["Envoy Sidecar"] --> AppB["App"]
    end

    EA <-->|mTLS| EB
```

*localhost:8080 → localhost:15001*

**Ventajas**:

- Aislamiento por workload
- L7 completo por pod

**Desventajas**:

- Un Envoy por pod = alto overhead
- Mucha memoria/CPU en clusters grandes

### 3.2 Node Proxy (Ambient Mode)

Un proxy por nodo para todos los pods:

```mermaid
flowchart TB
    subgraph Node["Node"]
        ZT["ztunnel<br/>(node proxy)<br/>← Un proxy por nodo"]
        ZT --> PA["Pod A"]
        ZT --> PB["Pod B"]
        ZT --> PC["Pod C"]
    end

    Note["← Sin sidecars"]
```

**Ventajas**:

- Mucho menor overhead (1 ztunnel por nodo vs N sidecars)
- L4 eficiente para todos

**Desventajas**:

- Solo L4 (necesita waypoint para L7)

---

## 4. Funciones Principales de un Proxy

### 4.1 Load Balancing

Distribuir tráfico entre múltiples backends:

```mermaid
flowchart LR
    C["Client"] --> P["Proxy<br/>(selecciona)"]
    P -->|25%| B1["Backend 1"]
    P -->|25%| B2["Backend 2"]
    P -->|25%| B3["Backend 3"]
    P -->|25%| B4["Backend 4"]
```

**Algoritmos**:

- **Round Robin**: Rotación circular
- **Least Connections**: Al backend con menos conexiones activas
- **Random**: Selección aleatoria
- **Ring Hash**: Consistent hashing (útil para caché)
- **Weighted**: Por pesos asignados

**En Envoy**:

```
source/common/upstream/load_balancer_impl.cc
```

### 4.2 Health Checking

Verificar que los backends estén sanos:

```mermaid
flowchart LR
    P["Proxy"] -->|Health Check| B1["Backend 1: ✓ Healthy"]
    P -->|Health Check| B2["Backend 2: ✗ Unhealthy<br/>(excluido)"]
    P -->|Health Check| B3["Backend 3: ✓ Healthy"]
```

*Tráfico solo va a 1 y 3*

**Tipos**:

- **Active**: El proxy envía requests de prueba
- **Passive**: Detecta fallos en tráfico real (outlier detection)

**En Envoy**:

```
source/common/upstream/health_checker_impl.cc
```

### 4.3 Circuit Breaking

Proteger backends sobrecargados:

```mermaid
stateDiagram-v2
    [*] --> CLOSED

    CLOSED: Proxy envía requests<br/>(funcionando)
    OPEN: Proxy corta tráfico<br/>devuelve 503<br/>(fallos excedidos)
    HALF_OPEN: Proxy envía 1 request<br/>de prueba<br/>(probando)

    CLOSED --> OPEN : errores > threshold
    OPEN --> HALF_OPEN : timer
    HALF_OPEN --> CLOSED : éxito
    HALF_OPEN --> OPEN : fallo
```

### 4.4 Retry y Timeout

Manejo de fallos transitorios:

```yaml
# Configuración Envoy
route_config:
  virtual_hosts:
    - routes:
        - route:
            timeout: 30s # Timeout total
            retry_policy:
              retry_on: "5xx,reset"
              num_retries: 3
              per_try_timeout: 10s
```

### 4.5 Observabilidad

```mermaid
flowchart TB
    C["Client"] --> P

    subgraph P["Proxy"]
        M["Metrics<br/>latency, rps, errors"]
        L["Logs<br/>access logs"]
        T["Traces<br/>span_id, trace_id"]
    end

    M --> Prom["Prometheus"]
    L --> Flu["Fluentd"]
    T --> Jae["Jaeger"]
```

---

## 5. Envoy vs ztunnel: Posicionamiento

```mermaid
flowchart LR
    subgraph Spectrum["Espectro de Proxies"]
        direction LR
        subgraph ZT["ztunnel<br/>Simpler/Faster<br/>L4 Only"]
            ZTF["- mTLS<br/>- L4 auth<br/>- Metrics<br/><br/>Node-level<br/>Transparent"]
        end

        subgraph ENV["Envoy<br/>Richer Features<br/>L4 + L7"]
            ENVF["- mTLS<br/>- L7 routing<br/>- JWT auth<br/>- Rate limiting<br/>- gRPC transcoding<br/>- WASM<br/><br/>Sidecar o Waypoint"]
        end

        ZT ---|◄────────────►| ENV
    end
```

---

## 6. Ejercicio de Reflexión

### Pregunta 1

Una empresa necesita:

- Load balancing entre 10 backends
- Health checks
- Métricas de latencia
- Sin modificar el código de las aplicaciones

¿Qué tipo de proxy usarían y por qué?

<details>
<summary>Respuesta</summary>

Un **reverse proxy** porque:

- Transparente para las aplicaciones (no necesitan código)
- Centraliza load balancing y health checks
- Puede exportar métricas sin cambios en apps
- Ejemplos: Nginx, HAProxy, Envoy

</details>

### Pregunta 2

Un cluster Kubernetes tiene 500 pods y necesita mTLS entre todos. ¿Qué modelo es más eficiente?

<details>
<summary>Respuesta</summary>

**Node proxy (ztunnel)** porque:

- 500 sidecars = alto overhead de memoria
- Si hay 50 nodos, solo 50 ztunnels
- Para solo mTLS, L4 es suficiente
- L7 (waypoint) solo donde se necesite

</details>

---

## 7. Autoevaluación

1. ¿Cuál es la diferencia entre forward y reverse proxy?
2. ¿Qué es un transparent proxy y cómo funciona?
3. ¿Por qué un service mesh usa proxies?
4. ¿Qué ventaja tiene el modelo node proxy sobre sidecars?
5. Nombra 3 funciones que realiza un proxy.

---

**Siguiente**: [02_proxy_l4.md](02_proxy_l4.md) - Proxy Layer 4 en Profundidad
