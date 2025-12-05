# L4 vs L7: Cuándo Usar Cada Uno

---

**Módulo**: 2 - Proxies - Conceptos Fundamentales
**Tema**: Comparativa L4 vs L7
**Tiempo estimado**: 1 hora
**Prerrequisitos**: [02_proxy_l4.md](02_proxy_l4.md), [03_proxy_l7.md](03_proxy_l7.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Podrás decidir cuándo usar L4 vs L7
- Entenderás el modelo híbrido de Istio ambient
- Tendrás un framework de decisión claro

---

## 1. Tabla Comparativa Completa

| Aspecto                    | Proxy L4               | Proxy L7                   |
| -------------------------- | ---------------------- | -------------------------- |
| **Capa OSI**               | Transport (4)          | Application (7)            |
| **Qué ve**                 | IP, puertos, TCP flags | + HTTP headers, body, gRPC |
| **Routing basado en**      | IP:puerto destino      | URL, headers, query params |
| **Ejemplos**               | ztunnel, TCP proxy     | Envoy HCM, Nginx           |
| **Latencia adicional**     | ~0.1ms                 | ~1-5ms                     |
| **Memoria por conexión**   | ~1KB                   | ~10-100KB                  |
| **CPU por request**        | Mínimo                 | Parsing + filters          |
| **mTLS**                   | ✓                      | ✓                          |
| **L4 Auth (IP, identity)** | ✓                      | ✓                          |
| **L7 Auth (JWT, OAuth)**   | ✗                      | ✓                          |
| **Rate limit por path**    | ✗                      | ✓                          |
| **Header manipulation**    | ✗                      | ✓                          |
| **Request-level metrics**  | ✗                      | ✓                          |
| **Protocolos custom**      | ✓ (bytes opacos)       | Solo si hay codec          |

---

## 2. Framework de Decisión

```mermaid
flowchart TB
    Q1["¿Qué proxy necesitas?"]
    Q2["¿Necesitas routing basado en<br/>URL, headers, o contenido?"]
    Q3["¿Necesitas validar JWT,<br/>OAuth, o API keys?"]
    Q4["¿Solo necesitas<br/>mTLS y L4 auth?"]

    L7A["Usa Proxy L7<br/>(Envoy HCM)"]
    L7B["Usa Proxy L7<br/>(Envoy HCM)"]
    L4["Usa Proxy L4<br/>(ztunnel)"]
    Eval["Evalúa caso<br/>específico"]

    Q1 --> Q2
    Q2 -->|"SÍ"| L7A
    Q2 -->|"NO"| Q3
    Q3 -->|"SÍ"| L7B
    Q3 -->|"NO"| Q4
    Q4 -->|"SÍ"| L4
    Q4 -->|"NO"| Eval

    style L7A fill:#4a9eff,color:#fff
    style L7B fill:#4a9eff,color:#fff
    style L4 fill:#10b981,color:#fff
    style Eval fill:#f59e0b,color:#fff
```

---

## 3. Casos de Uso Concretos

### 3.1 Caso: API Gateway

**Requisitos**:

- Routing por path (`/users`, `/orders`)
- JWT validation
- Rate limiting por API key
- Request logging con path

**Decisión**: **L7 (Envoy)**

```yaml
# Solo L7 puede hacer esto
route_config:
  routes:
    - match: { prefix: "/users" }
      route: { cluster: users-service }
    - match: { prefix: "/orders" }
      route: { cluster: orders-service }
```

### 3.2 Caso: mTLS Transparente en Kubernetes

**Requisitos**:

- Cifrado entre todos los pods
- Zero-trust networking
- Sin modificar aplicaciones
- 1000+ pods

**Decisión**: **L4 (ztunnel)**

```
Razones:
- mTLS no requiere L7
- 1000 sidecars vs 50 ztunnels (50 nodos)
- Menor latencia
- Menor uso de recursos
```

### 3.3 Caso: Service Mesh con Políticas de Seguridad

**Requisitos**:

- mTLS entre servicios
- Rate limiting por endpoint
- JWT validation
- Canary deployments basados en header

**Decisión**: **Híbrido (ztunnel + Waypoint)**

```
ztunnel: mTLS para todo
Waypoint (Envoy): L7 policies solo donde se necesitan
```

### 3.4 Caso: Proxy para Base de Datos

**Requisitos**:

- Proxy para MySQL
- Load balancing entre réplicas
- Connection pooling

**Decisión**: **L4**

```
Razones:
- Envoy no tiene codec MySQL nativo
- L4 funciona con cualquier protocolo
- No se necesita inspeccionar queries
```

---

## 4. El Modelo Híbrido: Istio Ambient

Istio ambient combina lo mejor de ambos mundos:

```mermaid
flowchart TB
    subgraph Ambient["Istio Ambient Mode"]
        subgraph L4Layer["Capa L4 (siempre activa)"]
            ZT["ztunnel (node proxy)"]
            ZTF["• mTLS automático<br/>• Identity (SPIFFE)<br/>• L4 authorization policies<br/>• Basic telemetry (connections, bytes)"]
        end

        subgraph L7Layer["Capa L7 (opcional, por namespace/servicio)"]
            WP["Waypoint Proxy (Envoy)"]
            WPF["• L7 authorization (path, headers)<br/>• JWT validation<br/>• Rate limiting<br/>• Traffic management (retries, timeouts)<br/>• Rich telemetry (per-path metrics)"]
        end
    end

    style L4Layer fill:#10b981,color:#fff
    style L7Layer fill:#4a9eff,color:#fff
```

### 4.1 Flujo de Tráfico en Ambient

```mermaid
flowchart LR
    subgraph NoWaypoint["Sin Waypoint (~0.2ms adicional)"]
        direction LR
        PA1["Pod A"] --> ZA1["ztunnel A"]
        ZA1 -->|"HBONE/mTLS"| ZB1["ztunnel B"]
        ZB1 --> PB1["Pod B"]
    end
```

```mermaid
flowchart LR
    subgraph WithWaypoint["Con Waypoint (~2-5ms adicional)"]
        direction LR
        PA2["Pod A"] --> ZA2["ztunnel A"]
        ZA2 --> WP["Waypoint<br/>(L7 features)"]
        WP --> ZB2["ztunnel B"]
        ZB2 --> PB2["Pod B"]
    end

    style WP fill:#4a9eff,color:#fff
```

### 4.2 Cuándo Usar Waypoint

| Servicio Necesita        | Sin Waypoint | Con Waypoint |
| ------------------------ | ------------ | ------------ |
| Solo mTLS                | ✓            | Overkill     |
| L4 auth (identity-based) | ✓            | Overkill     |
| L7 auth (path-based)     | ✗            | ✓            |
| JWT validation           | ✗            | ✓            |
| Rate limiting por path   | ✗            | ✓            |
| Canary por header        | ✗            | ✓            |

---

## 5. Comparación de Recursos

### 5.1 Memoria

**Cluster: 100 pods, 10 nodos**

| Modelo | Componentes | RAM Total |
|--------|-------------|-----------|
| **Sidecar (L7)** | 100 Envoy sidecars × ~50MB | ~5GB |
| **Ambient (L4)** | 10 ztunnels × ~30MB + Waypoints según necesidad | ~300MB + |

**Ahorro: >90% para baseline**

### 5.2 Latencia

**Latencia P99 adicional:**

| Componente | Latencia |
|------------|----------|
| Solo TCP proxy | ~0.1ms |
| ztunnel (HBONE) | ~0.2-0.5ms |
| Envoy sidecar (L7) | ~1-3ms |
| Waypoint (L7) | ~2-5ms |

> **Nota**: Para servicios sensibles a latencia, L4 puede ser la diferencia entre cumplir o no un SLA de 10ms.

---

## 6. Resumen de Decisión

```mermaid
flowchart TB
    subgraph Guide["Guía Rápida"]
        subgraph L4Use["Usa L4 (ztunnel) cuando:"]
            L4a["✓ Solo necesitas mTLS"]
            L4b["✓ Auth basado en identity (SPIFFE)"]
            L4c["✓ Protocolos no-HTTP"]
            L4d["✓ Latencia es crítica"]
            L4e["✓ Tienes muchos pods (escala)"]
        end

        subgraph L7Use["Usa L7 (Envoy) cuando:"]
            L7a["✓ Necesitas routing por URL/headers"]
            L7b["✓ JWT/OAuth validation"]
            L7c["✓ Rate limiting granular"]
            L7d["✓ Request/response transformation"]
            L7e["✓ Métricas por endpoint"]
        end

        subgraph Hybrid["Usa híbrido (ztunnel + Waypoint) cuando:"]
            Ha["✓ Baseline de mTLS para todo"]
            Hb["✓ L7 solo para servicios que lo necesitan"]
            Hc["✓ Quieres optimizar recursos"]
        end
    end

    style L4Use fill:#10b981,color:#fff
    style L7Use fill:#4a9eff,color:#fff
    style Hybrid fill:#8b5cf6,color:#fff
```

---

## 7. Autoevaluación

1. Un servicio solo necesita mTLS. ¿L4 o L7?
2. Necesitas rate limiting de 100 req/min para `/api/expensive`. ¿L4 o L7?
3. ¿Por qué Istio ambient usa ztunnel en lugar de Envoy sidecars?
4. ¿Cuándo agregarías un Waypoint en ambient mode?
5. ¿Qué modelo usarías para un cluster de 5000 pods?

---

**Siguiente Módulo**: [../03_envoy_arquitectura/01_vision_general.md](../03_envoy_arquitectura/01_vision_general.md) - Arquitectura de Envoy
