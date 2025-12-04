# Istio Ambient Mode: Arquitectura Completa

---

**Módulo**: 5 - Comparativa
**Tema**: Arquitectura Ambient Mode end-to-end
**Tiempo estimado**: 3 horas
**Prerrequisitos**: [01_envoy_vs_ztunnel.md](01_envoy_vs_ztunnel.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás la arquitectura completa de ambient mode
- Conocerás todos los componentes y su interacción
- Comprenderás los flujos de tráfico en diferentes escenarios
- Sabrás cómo configurar ambient mode en un cluster

---

## 1. Evolución del Service Mesh

### 1.1 Del Sidecar a Ambient

```
┌─────────────────────────────────────────────────────────────────┐
│                    Evolución de Istio                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2017-2022: SIDECAR MODE                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  ┌──────────────────┐    ┌──────────────────┐           │   │
│  │  │ Pod A            │    │ Pod B            │           │   │
│  │  │ ┌────┐ ┌───────┐ │    │ ┌────┐ ┌───────┐ │           │   │
│  │  │ │App │ │ Envoy │ │────│ │App │ │ Envoy │ │           │   │
│  │  │ └────┘ └───────┘ │    │ └────┘ └───────┘ │           │   │
│  │  └──────────────────┘    └──────────────────┘           │   │
│  │                                                          │   │
│  │  Características:                                        │   │
│  │  • Un Envoy por pod                                      │   │
│  │  • L4 + L7 siempre                                       │   │
│  │  • Alto overhead de recursos                             │   │
│  │  • Restart de pods para upgrades                         │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2022+: AMBIENT MODE                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │                     ztunnel                         │ │   │
│  │  │               (uno por nodo, L4)                    │ │   │
│  │  └────────────────────────┬───────────────────────────┘ │   │
│  │                           │                              │   │
│  │       ┌───────────────────┼───────────────────┐         │   │
│  │       ▼                   ▼                   ▼         │   │
│  │  ┌─────────┐         ┌─────────┐         ┌─────────┐   │   │
│  │  │ Pod A   │         │ Pod B   │         │ Pod C   │   │   │
│  │  │ (solo   │         │ (solo   │         │ (solo   │   │   │
│  │  │  app)   │         │  app)   │         │  app)   │   │   │
│  │  └─────────┘         └─────────┘         └─────────┘   │   │
│  │                                                          │   │
│  │  + Waypoint (opcional, para L7)                         │   │
│  │                                                          │   │
│  │  Características:                                        │   │
│  │  • Un ztunnel por nodo (no por pod)                     │   │
│  │  • L4 siempre, L7 opcional                              │   │
│  │  • Bajo overhead de recursos                            │   │
│  │  • No restart de pods para upgrades                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Motivación de Ambient

| Problema en Sidecar        | Solución en Ambient           |
| -------------------------- | ----------------------------- |
| RAM: 50-100MB por pod      | ztunnel: ~50MB por nodo       |
| Upgrade = restart pods     | ztunnel upgrade independiente |
| L7 siempre (overhead)      | L7 solo donde se necesita     |
| Inyección mutating webhook | Sin inyección                 |
| Complejidad de debugging   | Menos componentes             |

---

## 2. Componentes de Ambient Mode

### 2.1 Arquitectura de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                  Ambient Mode Components                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                     CONTROL PLANE                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                        istiod                            │   │
│  │  ┌─────────────────┐  ┌─────────────────┐               │   │
│  │  │  xDS Server     │  │  CA (Citadel)   │               │   │
│  │  │  (config push)  │  │  (certs)        │               │   │
│  │  └────────┬────────┘  └────────┬────────┘               │   │
│  │           │                    │                         │   │
│  └───────────┼────────────────────┼─────────────────────────┘   │
│              │                    │                             │
│              │    xDS + SDS       │                             │
│              │                    │                             │
│              ▼                    ▼                             │
│                      DATA PLANE                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │                   ztunnel                        │    │   │
│  │  │                (DaemonSet)                       │    │   │
│  │  │                                                  │    │   │
│  │  │  • Recibe config via xDS                        │    │   │
│  │  │  • Recibe certs via SDS                         │    │   │
│  │  │  • Aplica mTLS a todo el tráfico               │    │   │
│  │  │  • Aplica L4 policies                          │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │               Waypoint Proxy                     │    │   │
│  │  │              (Deployment, opcional)              │    │   │
│  │  │                                                  │    │   │
│  │  │  • Envoy configurado para L7                    │    │   │
│  │  │  • Por namespace o servicio                     │    │   │
│  │  │  • HTTP routing, JWT, rate limiting             │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │                 istio-cni                        │    │   │
│  │  │               (DaemonSet)                        │    │   │
│  │  │                                                  │    │   │
│  │  │  • Configura iptables en pods                   │    │   │
│  │  │  • Redirección de tráfico a ztunnel             │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Detalle de Cada Componente

#### istiod (Control Plane)

```yaml
# Deployment de istiod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: istiod
  namespace: istio-system
spec:
  replicas: 1 # O más para HA
  template:
    spec:
      containers:
        - name: discovery
          image: istio/pilot:latest
          ports:
            - containerPort: 15010 # xDS gRPC
            - containerPort: 15012 # xDS mTLS
            - containerPort: 15014 # Control plane monitoring
            - containerPort: 15017 # Webhook/injection

# Funciones:
# • Pilot: xDS server (LDS, CDS, EDS, RDS)
# • Citadel: Certificate Authority
# • Galley: Config validation
```

#### ztunnel (Node Proxy)

```yaml
# DaemonSet de ztunnel
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ztunnel
  namespace: istio-system
spec:
  template:
    spec:
      hostNetwork: true # Puede necesitarlo para algunos modos
      containers:
        - name: ztunnel
          image: istio/ztunnel:latest
          securityContext:
            capabilities:
              add:
                - NET_ADMIN
                - SYS_ADMIN
          ports:
            - containerPort: 15001 # Outbound capture
            - containerPort: 15006 # Inbound plaintext
            - containerPort: 15008 # HBONE
            - containerPort: 15020 # Metrics
            - containerPort: 15021 # Readiness
```

#### Waypoint Proxy (L7 opcional)

```yaml
# Deployment de Waypoint
apiVersion: apps/v1
kind: Deployment
metadata:
  name: waypoint
  namespace: default # O namespace específico
  labels:
    istio.io/gateway-name: waypoint
spec:
  template:
    spec:
      containers:
        - name: istio-proxy
          image: istio/proxyv2:latest # Envoy
          ports:
            - containerPort: 15008 # HBONE inbound
            - containerPort: 15001 # Outbound
```

#### istio-cni (Traffic Redirect)

```yaml
# DaemonSet de istio-cni
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: istio-cni-node
  namespace: istio-system
spec:
  template:
    spec:
      containers:
        - name: install-cni
          # Instala plugin CNI
        - name: istio-cni
          # Configura redirección iptables
```

---

## 3. Flujos de Tráfico

### 3.1 L4 Only (Sin Waypoint)

```
┌─────────────────────────────────────────────────────────────────┐
│              Flujo L4 Only (ztunnel a ztunnel)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Node A                                    Node B               │
│  ┌───────────────────────────────┐  ┌───────────────────────┐  │
│  │                                │  │                       │  │
│  │  Pod (client)                 │  │  Pod (server)         │  │
│  │  ┌─────────────────────────┐  │  │  ┌─────────────────┐  │  │
│  │  │ connect(svc-b:8080)     │  │  │  │ listen(:8080)   │  │  │
│  │  └───────────┬─────────────┘  │  │  └────────▲────────┘  │  │
│  │              │ ①               │  │           │ ⑥         │  │
│  │              ▼                 │  │           │           │  │
│  │  ┌─────────────────────────┐  │  │  ┌────────┴────────┐  │  │
│  │  │ iptables → 15001        │  │  │  │ ztunnel B       │  │  │
│  │  └───────────┬─────────────┘  │  │  │ :15008 HBONE    │  │  │
│  │              │ ②               │  │  └────────▲────────┘  │  │
│  │              ▼                 │  │           │           │  │
│  │  ┌─────────────────────────┐  │  │           │ ⑤         │  │
│  │  │ ztunnel A               │  │  │           │           │  │
│  │  │ :15001 outbound         │  │  │           │           │  │
│  │  │                         │  │  │           │           │  │
│  │  │ ③ mTLS handshake        │  │  │           │           │  │
│  │  │   SPIFFE cert client    │──┼──┼───────────┘           │  │
│  │  │                         │  │  │                       │  │
│  │  │ ④ HBONE CONNECT         │  │  │                       │  │
│  │  │   :authority=svc-b:8080 │──┼──┼──> (via network)      │  │
│  │  │                         │  │  │                       │  │
│  │  └─────────────────────────┘  │  │                       │  │
│  │                                │  │                       │  │
│  └───────────────────────────────┘  └───────────────────────┘  │
│                                                                 │
│  Pasos:                                                        │
│  ① App llama connect()                                         │
│  ② iptables redirige a ztunnel:15001                          │
│  ③ ztunnel A inicia mTLS con cert SPIFFE del client           │
│  ④ ztunnel A envía HBONE CONNECT a ztunnel B:15008            │
│  ⑤ ztunnel B recibe HBONE, verifica cert, extrae destino      │
│  ⑥ ztunnel B conecta al pod servidor                          │
│                                                                 │
│  Características:                                              │
│  • mTLS automático (zero trust)                                │
│  • Sin parsing HTTP                                            │
│  • Baja latencia                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Con Waypoint (L7 Features)

```
┌─────────────────────────────────────────────────────────────────┐
│              Flujo con Waypoint (L4 + L7)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │  Pod (client)                                              ││
│  │  ┌─────────────────────────────┐                           ││
│  │  │ GET /api/users              │                           ││
│  │  │ Authorization: Bearer xxx   │                           ││
│  │  └───────────┬─────────────────┘                           ││
│  │              │                                              ││
│  │              ▼                                              ││
│  │  ┌─────────────────────────────┐                           ││
│  │  │ ztunnel (source node)       │                           ││
│  │  │ • Detecta que destino tiene │                           ││
│  │  │   waypoint configurado     │                           ││
│  │  │ • Envía a waypoint primero  │                           ││
│  │  └───────────┬─────────────────┘                           ││
│  │              │                                              ││
│  └──────────────┼─────────────────────────────────────────────┘│
│                 │ HBONE (mTLS)                                  │
│                 ▼                                               │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  Waypoint Proxy (Envoy)                                    ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │ L7 Processing:                                       │   ││
│  │  │ ① Termina HTTP                                       │   ││
│  │  │ ② Valida JWT token                                   │   ││
│  │  │ ③ Aplica rate limiting                               │   ││
│  │  │ ④ Enruta según path                                  │   ││
│  │  │ ⑤ Aplica AuthorizationPolicy                        │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  │              │                                              ││
│  └──────────────┼─────────────────────────────────────────────┘│
│                 │ HBONE (mTLS)                                  │
│                 ▼                                               │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  ztunnel (destination node)                                ││
│  │  ┌─────────────────────────────┐                           ││
│  │  │ Recibe HBONE               │                           ││
│  │  │ Conecta a pod destino      │                           ││
│  │  └───────────┬─────────────────┘                           ││
│  │              │                                              ││
│  │              ▼                                              ││
│  │  ┌─────────────────────────────┐                           ││
│  │  │ Pod (server)                │                           ││
│  │  │ :8080                       │                           ││
│  │  └─────────────────────────────┘                           ││
│  │                                                             ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
│  Flujo: client → ztunnel → waypoint → ztunnel → server        │
│                    L4         L7         L4                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Intra-Node (Mismo Nodo)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Flujo Intra-Node                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Node                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Pod A (client)              Pod B (server)              │   │
│  │  ┌─────────────┐            ┌─────────────┐             │   │
│  │  │ connect()   │            │ listen()    │             │   │
│  │  └──────┬──────┘            └──────▲──────┘             │   │
│  │         │                          │                     │   │
│  │         │ ①                        │ ⑤                   │   │
│  │         ▼                          │                     │   │
│  │  ┌───────────────────────────────────────────────────┐  │   │
│  │  │                     ztunnel                        │  │   │
│  │  │                                                    │  │   │
│  │  │  ② Recibe en :15001 (outbound)                    │  │   │
│  │  │  ③ Detecta que destino está en mismo nodo         │  │   │
│  │  │  ④ Conecta directo (o via loopback HBONE)         │  │   │
│  │  │                                                    │  │   │
│  │  └───────────────────────────────────────────────────┘  │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Optimización:                                                 │
│  • Si mismo nodo, puede evitar HBONE completo                  │
│  • mTLS sigue aplicándose                                      │
│  • Menor latencia que inter-node                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Configuración de Ambient Mode

### 4.1 Instalación de Istio con Ambient

```bash
# Instalar Istio con perfil ambient
istioctl install --set profile=ambient

# O con Helm
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system
helm install istio-cni istio/cni -n istio-system
helm install ztunnel istio/ztunnel -n istio-system
```

### 4.2 Habilitar Ambient para un Namespace

```bash
# Añadir label al namespace
kubectl label namespace default istio.io/dataplane-mode=ambient

# Verificar
kubectl get namespace default --show-labels
```

### 4.3 Crear Waypoint Proxy

```bash
# Crear waypoint para namespace
istioctl waypoint apply -n default

# O para un servicio específico
istioctl waypoint apply -n default --name my-service-waypoint \
  --for service/my-service
```

```yaml
# También puede hacerse con Gateway API
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  namespace: default
  labels:
    istio.io/waypoint-for: service
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

### 4.4 Configurar Políticas L7 (requiere Waypoint)

```yaml
# AuthorizationPolicy que requiere L7
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: default
spec:
  targetRefs:
    - kind: Service
      name: my-api
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
  action: ALLOW
---
# RequestAuthentication para JWT
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: default
spec:
  targetRefs:
    - kind: Service
      name: my-api
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

---

## 5. Observabilidad en Ambient

### 5.1 Métricas

```
┌─────────────────────────────────────────────────────────────────┐
│                    Metrics Collection                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ztunnel Metrics (port 15020):                                 │
│  ─────────────────────────────                                  │
│  • istio_tcp_connections_opened_total                          │
│  • istio_tcp_connections_closed_total                          │
│  • istio_tcp_sent_bytes_total                                  │
│  • istio_tcp_received_bytes_total                              │
│  • istio_build (version info)                                  │
│                                                                 │
│  Waypoint Metrics (si habilitado):                             │
│  ─────────────────────────────────                              │
│  • istio_requests_total                                        │
│  • istio_request_duration_milliseconds                         │
│  • istio_request_bytes                                         │
│  • istio_response_bytes                                        │
│  • + todas las métricas L7 de Envoy                            │
│                                                                 │
│  Prometheus config:                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  - job_name: 'ztunnel'                                   │  │
│  │    kubernetes_sd_configs:                                 │  │
│  │    - role: pod                                           │  │
│  │    relabel_configs:                                       │  │
│  │    - source_labels: [__meta_kubernetes_pod_label_app]    │  │
│  │      regex: ztunnel                                       │  │
│  │      action: keep                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Logs

```bash
# ztunnel logs
kubectl logs -n istio-system -l app=ztunnel -f

# Con más detalle
kubectl logs -n istio-system -l app=ztunnel -f | grep -E "(CONNECT|connection)"

# Waypoint logs (si existe)
kubectl logs -n default -l gateway.networking.k8s.io/gateway-name=waypoint -f
```

### 5.3 Debugging

```bash
# Ver estado de ztunnel
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=ztunnel -o name | head -1) -- curl localhost:15000/config_dump

# Ver workloads conocidos por ztunnel
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=ztunnel -o name | head -1) -- curl localhost:15000/debug/workloads

# Proxy-status para ambient
istioctl proxy-status

# Ver config de un workload
istioctl pc workload <pod-name>.<namespace>
```

---

## 6. Seguridad en Ambient

### 6.1 mTLS Automático

```
┌─────────────────────────────────────────────────────────────────┐
│                      mTLS en Ambient                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Certificados SPIFFE:                                          │
│  ─────────────────────                                          │
│  • istiod actúa como CA                                        │
│  • ztunnel obtiene certs para CADA workload en su nodo         │
│  • Certs rotan automáticamente (24h default)                   │
│                                                                 │
│  Identidad:                                                    │
│  ──────────                                                     │
│  spiffe://cluster.local/ns/<namespace>/sa/<service-account>   │
│                                                                 │
│  Flujo de certificados:                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  istiod (CA)                                            │   │
│  │      │                                                   │   │
│  │      │ SDS (Secret Discovery Service)                   │   │
│  │      ▼                                                   │   │
│  │  ztunnel                                                │   │
│  │      │                                                   │   │
│  │      │ Usa cert según workload                          │   │
│  │      │ source → cert de source SA                       │   │
│  │      │ dest   → verifica cert de dest SA                │   │
│  │      ▼                                                   │   │
│  │  [mTLS connection con identidades correctas]            │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Authorization Policies

```yaml
# L4 Authorization (ztunnel aplica)
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  selector:
    matchLabels:
      app: backend
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/frontend"]
  action: ALLOW
---
# L7 Authorization (requiere waypoint)
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-valid-paths
  namespace: default
spec:
  targetRefs:
    - kind: Service
      name: api-service
  rules:
    - to:
        - operation:
            paths: ["/api/*"]
            methods: ["GET", "POST"]
  action: ALLOW
```

---

## 7. Limitaciones y Consideraciones

### 7.1 Limitaciones Actuales

```
┌─────────────────────────────────────────────────────────────────┐
│                    Limitaciones de Ambient                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Funcionalidad:                                                │
│  ──────────────                                                 │
│  • No todas las features de sidecar disponibles               │
│  • Algunas EnvoyFilters no aplican                            │
│  • WASM solo en waypoint                                       │
│                                                                 │
│  Networking:                                                   │
│  ───────────                                                    │
│  • Requiere kernel relativamente moderno                       │
│  • Algunos CNIs pueden tener incompatibilidades               │
│  • HostNetwork pods tienen limitaciones                        │
│                                                                 │
│  Madurez:                                                      │
│  ────────                                                       │
│  • Ambient es más nuevo que sidecar mode                       │
│  • Menos batalla-probado en producción                         │
│  • Documentación aún evolucionando                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Cuándo NO Usar Ambient

| Escenario                       | Por qué                        |
| ------------------------------- | ------------------------------ |
| **Todo el tráfico necesita L7** | Overhead de waypoint para todo |
| **WASM extensivo**              | Solo soportado en waypoint     |
| **Compliance estricto**         | Sidecars más maduros           |
| **Ciertos CNIs**                | Verificar compatibilidad       |

### 7.3 Cuándo Usar Ambient

| Escenario               | Por qué                     |
| ----------------------- | --------------------------- |
| **Mayoría L4, poco L7** | Optimal resource usage      |
| **Clusters grandes**    | Ahorro significativo de RAM |
| **Upgrades frecuentes** | Sin restart de pods         |
| **Simplicidad**         | Menos componentes por pod   |

---

## 8. Migración de Sidecar a Ambient

### 8.1 Estrategia de Migración

```
┌─────────────────────────────────────────────────────────────────┐
│                    Migration Strategy                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Fase 1: Preparación                                           │
│  ─────────────────────                                          │
│  • Verificar compatibilidad de CNI                             │
│  • Revisar policies existentes                                 │
│  • Identificar servicios que requieren L7                      │
│                                                                 │
│  Fase 2: Piloto                                                │
│  ───────────────                                                │
│  • Elegir namespace no crítico                                 │
│  • Quitar sidecar injection                                    │
│  • Habilitar ambient mode                                      │
│  • Añadir waypoint si necesario                                │
│  • Verificar funcionamiento                                    │
│                                                                 │
│  Fase 3: Rollout                                               │
│  ─────────────────                                              │
│  • Namespace por namespace                                     │
│  • Monitorear métricas                                         │
│  • Rollback plan listo                                         │
│                                                                 │
│  Fase 4: Cleanup                                               │
│  ────────────────                                               │
│  • Quitar injection labels                                     │
│  • Verificar pods sin sidecars                                 │
│  • Actualizar documentación                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Comandos de Migración

```bash
# 1. Quitar sidecar injection del namespace
kubectl label namespace default istio-injection-

# 2. Habilitar ambient
kubectl label namespace default istio.io/dataplane-mode=ambient

# 3. Restart pods para quitar sidecars
kubectl rollout restart deployment -n default

# 4. Verificar
kubectl get pods -n default -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | grep -v istio-proxy
```

---

## 9. Autoevaluación

1. ¿Cuáles son los componentes principales de ambient mode?
2. ¿Cuándo se usa un waypoint proxy?
3. ¿Cómo fluye el tráfico entre dos pods en diferentes nodos?
4. ¿Qué tipo de policies puede aplicar ztunnel vs waypoint?
5. ¿Cuál es la principal ventaja de recursos de ambient?

---

## 10. Referencias

| Recurso                                                                    | Descripción            |
| -------------------------------------------------------------------------- | ---------------------- |
| [Istio Ambient Docs](https://istio.io/latest/docs/ambient/)                | Documentación oficial  |
| [Ambient Architecture](https://istio.io/latest/docs/ambient/architecture/) | Arquitectura detallada |
| [Getting Started](https://istio.io/latest/docs/ambient/getting-started/)   | Tutorial inicial       |
| [ztunnel repo](https://github.com/istio/ztunnel)                           | Código fuente          |

---

**Siguiente**: [03_decision_framework.md](03_decision_framework.md) - Framework de Decisión
