# Istio Ambient Mode y Contexto de ztunnel

---

**Módulo**: 4 - Arquitectura de ztunnel
**Tema**: Contexto de Ambient Mode
**Tiempo estimado**: 2 horas
**Prerrequisitos**: Módulos 1-3 completos

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás qué es Istio ambient mode
- Conocerás el rol de ztunnel en la arquitectura
- Comprenderás por qué se creó ztunnel
- Identificarás las diferencias con el modelo sidecar

---

## 1. El Problema con Sidecars

### 1.1 Modelo Sidecar Tradicional

```mermaid
flowchart TB
    subgraph Node1["Node 1 - Modelo Sidecar"]
        subgraph PodA["Pod A"]
            AppA["App"]
            SidecarA["Envoy Sidecar"]
        end

        subgraph PodB["Pod B"]
            AppB["App"]
            SidecarB["Envoy Sidecar"]
        end

        subgraph PodC["Pod C"]
            AppC["App"]
            SidecarC["Envoy Sidecar"]
        end

        subgraph PodD["Pod D"]
            AppD["App"]
            SidecarD["Envoy Sidecar"]
        end
    end

    Note["4 pods = 4 sidecars Envoy"]
```

### 1.2 Problemas del Modelo Sidecar

| Problema                   | Descripción                 | Impacto                             |
| -------------------------- | --------------------------- | ----------------------------------- |
| **Overhead de recursos**   | ~50-100MB RAM por sidecar   | Cluster de 1000 pods = 50-100GB RAM |
| **Latencia**               | Parsing L7 en cada hop      | +1-5ms por request                  |
| **Complejidad de upgrade** | Cada pod necesita restart   | Disruption durante updates          |
| **Startup time**           | Envoy inicia con el pod     | Pods lentos para arrancar           |
| **Seguridad**              | Sidecar tiene acceso al pod | Superficie de ataque mayor          |

### 1.3 La Pregunta Clave

> "¿Realmente TODOS los servicios necesitan L7?"

La respuesta: **No**. Muchos solo necesitan:

- mTLS (cifrado)
- Identity (autenticación)
- L4 authorization
- Métricas básicas

---

## 2. Ambient Mode: La Solución

### 2.1 Arquitectura de Dos Capas

```mermaid
flowchart TB
    subgraph Ambient["Ambient Mode Architecture"]
        subgraph L7Layer["LAYER 7 (cuando se necesita)"]
            Waypoint["Waypoint Proxy (Envoy)"]
            L7Features["• L7 routing, policies<br/>• JWT validation<br/>• Rate limiting<br/>• Traffic management<br/>• OPCIONAL - solo donde se necesita"]
        end

        subgraph L4Layer["LAYER 4 (siempre activo)"]
            Ztunnel["ztunnel (Node Proxy)"]
            L4Features["• mTLS automático<br/>• Identity (SPIFFE)<br/>• L4 authorization<br/>• Telemetry<br/>• SIEMPRE presente en cada nodo"]
        end

        L7Layer <--> L4Layer
    end

    style L7Layer fill:#4a9eff,color:#fff
    style L4Layer fill:#10b981,color:#fff
```

### 2.2 Comparación

| Aspecto             | Sidecar       | Ambient                   |
| ------------------- | ------------- | ------------------------- |
| **Proxies por pod** | 1 Envoy       | 0 (ztunnel en nodo)       |
| **RAM overhead**    | ~50-100MB/pod | ~30MB/nodo                |
| **L4 features**     | Siempre       | Siempre (ztunnel)         |
| **L7 features**     | Siempre       | Solo si waypoint          |
| **Upgrade**         | Restart pods  | Restart ztunnel DaemonSet |
| **Startup**         | Con cada pod  | Ya corriendo              |

---

## 3. ztunnel: El Node Proxy

### 3.1 ¿Qué es ztunnel?

**ztunnel** (Zero Trust Tunnel) es un proxy L4 escrito en Rust:

```
De README.md:
"Ztunnel is intended to be a purpose built implementation of the
node proxy in ambient mesh. Part of the goals of this included
keeping a narrow feature set."
```

### 3.2 Características Clave

| ✓ Incluido (In Scope) | ✗ Excluido (Out of Scope) |
|----------------------|---------------------------|
| mTLS between workloads | HTTP traffic termination |
| SPIFFE identity | Generic extensibility |
| L4 authorization | WASM, Lua, ext_authz |
| L4 telemetry | |
| HBONE tunneling | |

> "Ztunnel does not aim to be a generic extensible proxy; Envoy is better suited for that task."

### 3.3 Por qué Rust

| Razón              | Beneficio                            |
| ------------------ | ------------------------------------ |
| **Memory safety**  | Sin buffer overflows, use-after-free |
| **Performance**    | Comparable a C/C++                   |
| **Async native**   | Tokio runtime eficiente              |
| **Modern tooling** | Cargo, testing, etc.                 |
| **Security**       | Menos CVEs que C/C++                 |

---

## 4. Arquitectura de ztunnel

### 4.1 Despliegue

ztunnel corre como **DaemonSet** en Kubernetes:

```yaml
# Cada nodo tiene exactamente un ztunnel
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ztunnel
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: ztunnel
  template:
    spec:
      containers:
        - name: ztunnel
          image: istio/ztunnel:latest
          securityContext:
            capabilities:
              add:
                - NET_ADMIN # Para iptables
                - SYS_ADMIN # Para network namespaces
```

### 4.2 Flujo de Tráfico

```mermaid
flowchart TB
    subgraph Node["Node"]
        subgraph ZT["ztunnel"]
            subgraph Ports["Ports"]
                P15008["Port 15008<br/>(HBONE in)"]
                P15001["Port 15001<br/>(capture)"]
                P15006["Port 15006<br/>(plaintext)"]
            end
        end

        ZT --> PodA & PodB & PodC

        PodA["Pod A (app)"]
        PodB["Pod B (app)"]
        PodC["Pod C (app)"]

        Note["Las apps no saben que hay un proxy"]
    end
```

### 4.3 HBONE Protocol

HBONE (HTTP-Based Overlay Network Encapsulation):

**Formato:**
```
HTTP/2 CONNECT request
:method = CONNECT
:authority = 10.0.1.5:8080 (destino real)
:protocol = connect-tcp

Sobre mTLS con certificados SPIFFE
Puerto 15008
```

```mermaid
flowchart LR
    PodA["Pod A"] -->|"plaintext"| ZTA["ztunnel A"]
    ZTA -->|"HBONE/mTLS:15008"| ZTB["ztunnel B"]
    ZTB -->|"plaintext"| PodB["Pod B"]

    Note["El túnel HBONE transporta bytes opacos<br/>ztunnel NO parsea HTTP del usuario"]
```

---

## 5. Comparación con Envoy

### 5.1 Tabla Detallada

| Característica       | Envoy      | ztunnel            |
| -------------------- | ---------- | ------------------ |
| **Lenguaje**         | C++20      | Rust               |
| **Líneas de código** | ~500K      | ~30K               |
| **Capa**             | L4 + L7    | Solo L4            |
| **HTTP parsing**     | Sí         | No                 |
| **gRPC support**     | Sí         | No (payload opaco) |
| **Filter chain**     | Extensible | Mínimo             |
| **WASM**             | Sí         | No                 |
| **Memoria típica**   | 50-100MB   | 20-50MB            |
| **Startup time**     | ~1-2s      | ~100ms             |

### 5.2 Cuándo Usar Cada Uno

```mermaid
flowchart TB
    Start["Decision Tree"]
    Q1["¿Necesitas features L7?<br/>(routing por path, JWT, etc.)"]

    Start --> Q1

    Q1 -->|"SÍ"| Waypoint["Usa Waypoint (Envoy)<br/>L7 por servicio o namespace"]
    Q1 -->|"NO"| Ztunnel["Solo ztunnel es suficiente<br/>L4 para todo"]

    style Waypoint fill:#4a9eff,color:#fff
    style Ztunnel fill:#10b981,color:#fff
```

---

## 6. Habilitando Ambient Mode

### 6.1 Por Namespace

```bash
# Habilitar ambient para un namespace
kubectl label namespace default istio.io/dataplane-mode=ambient
```

### 6.2 Verificar Estado

```bash
# Ver pods en ambient mode
kubectl get pods -n default -o jsonpath='{.items[*].metadata.annotations.ambient\.istio\.io/redirection}'

# Ver ztunnel logs
kubectl logs -n istio-system -l app=ztunnel
```

---

## 7. Autoevaluación

1. ¿Cuál es el principal problema que resuelve ambient mode?
2. ¿Por qué ztunnel está escrito en Rust y no en C++?
3. ¿Qué features tiene ztunnel que NO tiene?
4. ¿Qué es HBONE y para qué sirve?
5. ¿Cuándo necesitas un waypoint proxy?

---

**Siguiente**: [02_threading_tokio.md](02_threading_tokio.md) - Threading Model con Tokio
