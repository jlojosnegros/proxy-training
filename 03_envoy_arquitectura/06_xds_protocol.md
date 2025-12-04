# xDS Protocol: Configuración Dinámica

---

**Módulo**: 3 - Arquitectura de Envoy
**Tema**: xDS Protocol
**Tiempo estimado**: 3 horas
**Prerrequisitos**: [05_cluster_management.md](05_cluster_management.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás el modelo xDS para configuración dinámica
- Conocerás los diferentes tipos de Discovery Services
- Comprenderás el flujo de suscripción y actualización
- Sabrás cómo Envoy recibe configuración de Istio

---

## 1. ¿Qué es xDS?

### 1.1 El Problema

Configuración estática tiene limitaciones:

- Requiere restart para cambios
- No escala a miles de endpoints
- No se integra con service discovery

### 1.2 La Solución: xDS

xDS (x Discovery Service) es un conjunto de APIs para configuración dinámica:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Control Plane                               │
│            (Istio, Consul, custom server)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │   LDS   │ │   RDS   │ │   CDS   │ │   EDS   │ │   SDS   │  │
│  │Listeners│ │ Routes  │ │Clusters │ │Endpoints│ │ Secrets │  │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘  │
│       │          │          │          │          │           │
└───────┼──────────┼──────────┼──────────┼──────────┼───────────┘
        │          │          │          │          │
        │          │   gRPC streaming (bi-directional)           │
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Envoy                                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │Listener │ │  Route  │ │ Cluster │ │Endpoints│ │  Certs  │  │
│  │ Config  │ │ Config  │ │ Config  │ │  List   │ │  Store  │  │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘  │
│                                                                 │
│  Configuración actualizada sin restart                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Tipos de xDS

### 2.1 Discovery Services

| API      | Nombre Completo                    | Qué Configura                      |
| -------- | ---------------------------------- | ---------------------------------- |
| **LDS**  | Listener Discovery Service         | Listeners (puertos, filter chains) |
| **RDS**  | Route Discovery Service            | Rutas HTTP (virtual hosts, paths)  |
| **CDS**  | Cluster Discovery Service          | Clusters (upstream groups)         |
| **EDS**  | Endpoint Discovery Service         | Endpoints (IPs de backends)        |
| **SDS**  | Secret Discovery Service           | Certificados TLS                   |
| **VHDS** | Virtual Host Discovery Service     | Virtual hosts dinámicos            |
| **SRDS** | Scoped Route Discovery Service     | Rutas con scope                    |
| **RTDS** | Runtime Discovery Service          | Runtime flags                      |
| **ECDS** | Extension Config Discovery Service | Configuración de extensiones       |

### 2.2 Orden de Dependencias

```
                    LDS
                     │
                     ▼
               ┌─────────┐
               │   RDS   │ (referenciado por listener)
               └────┬────┘
                    │
                    ▼
               ┌─────────┐
               │   CDS   │ (referenciado por routes)
               └────┬────┘
                    │
                    ▼
               ┌─────────┐
               │   EDS   │ (referenciado por clusters)
               └─────────┘

Un listener necesita rutas → rutas necesitan clusters → clusters necesitan endpoints
```

---

## 3. Protocolo de Comunicación

### 3.1 gRPC Streaming

```protobuf
// api/envoy/service/discovery/v3/ads.proto

service AggregatedDiscoveryService {
    // Stream bidireccional
    rpc StreamAggregatedResources(stream DiscoveryRequest)
        returns (stream DiscoveryResponse);

    rpc DeltaAggregatedResources(stream DeltaDiscoveryRequest)
        returns (stream DeltaDiscoveryResponse);
}
```

### 3.2 Flujo de Request/Response

```
Envoy                                          Control Plane
  │                                                  │
  │───── DiscoveryRequest ──────────────────────────>│
  │      type_url: "type.googleapis.com/...Listener" │
  │      node: { id: "envoy-1", cluster: "..." }     │
  │      resource_names: []  (wildcard)              │
  │                                                  │
  │<──── DiscoveryResponse ─────────────────────────│
  │      version_info: "v1"                          │
  │      resources: [Listener1, Listener2]           │
  │      nonce: "abc123"                             │
  │                                                  │
  │───── DiscoveryRequest (ACK) ────────────────────>│
  │      version_info: "v1"                          │
  │      response_nonce: "abc123"                    │
  │      (sin error_detail = ACK)                    │
  │                                                  │
  │      ... tiempo pasa ...                         │
  │                                                  │
  │<──── DiscoveryResponse (update) ────────────────│
  │      version_info: "v2"                          │
  │      resources: [Listener1, Listener2, Listener3]│
  │      nonce: "def456"                             │
  │                                                  │
  │───── DiscoveryRequest (ACK) ────────────────────>│
  │      version_info: "v2"                          │
  │      response_nonce: "def456"                    │
```

### 3.3 ACK vs NACK

```cpp
// ACK: Configuración aceptada
DiscoveryRequest {
    version_info: "v2"      // Versión que se aceptó
    response_nonce: "xyz"   // Nonce del response
    // Sin error_detail
}

// NACK: Configuración rechazada
DiscoveryRequest {
    version_info: "v1"      // Versión anterior (la que sigue activa)
    response_nonce: "xyz"   // Nonce del response rechazado
    error_detail: {
        code: INVALID_ARGUMENT
        message: "Invalid route configuration"
    }
}
```

---

## 4. Aggregated Discovery Service (ADS)

### 4.1 ¿Por qué ADS?

Sin ADS, cada tipo de recurso usa un stream separado:

- Posibles race conditions
- Configuración inconsistente temporal

Con ADS, todo va por un solo stream ordenado:

```yaml
# Bootstrap config para ADS
dynamic_resources:
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster

  lds_config:
    resource_api_version: V3
    ads: {} # Usar ADS

  cds_config:
    resource_api_version: V3
    ads: {} # Usar ADS
```

### 4.2 Orden de Entrega con ADS

```
Control Plane envía en orden:

1. CDS (clusters primero)
   └─► Envoy crea cluster objects

2. EDS (endpoints para clusters)
   └─► Envoy llena endpoints

3. LDS (listeners)
   └─► Envoy crea listeners

4. RDS (routes para listeners)
   └─► Envoy activa rutas

Esto evita referencias a clusters/rutas que no existen
```

---

## 5. Implementación en Envoy

### 5.1 Código Principal

```cpp
// source/common/config/grpc_mux_impl.cc

class GrpcMuxImpl : public GrpcMux {
    // Stream gRPC al control plane
    Grpc::AsyncClient::Stream<DiscoveryRequest, DiscoveryResponse> stream_;

    // Callbacks por tipo de recurso
    std::map<std::string, WatchMap> api_state_;

    void sendDiscoveryRequest(const std::string& type_url) {
        DiscoveryRequest request;
        request.set_type_url(type_url);
        request.mutable_node()->CopyFrom(node_);
        // ... llenar resource_names, version_info, etc.
        stream_->sendMessage(request);
    }

    void onReceiveMessage(std::unique_ptr<DiscoveryResponse>&& message) {
        // Procesar response
        for (const auto& resource : message->resources()) {
            // Decodificar y aplicar configuración
            applyConfiguration(resource);
        }
        // Enviar ACK
        sendDiscoveryRequest(message->type_url());
    }
};
```

### 5.2 Subscription Interface

```cpp
// source/common/config/subscription_impl.cc

class GrpcSubscriptionImpl : public Subscription {
public:
    // Crear suscripción a un tipo de recurso
    void start(const std::set<std::string>& resource_names) {
        grpc_mux_->subscribe(type_url_, resource_names, *this);
    }

    // Callback cuando llega configuración
    void onConfigUpdate(const std::vector<Config::DecodedResourceRef>& resources,
                        const std::string& version_info) {
        // Validar y aplicar
        for (const auto& resource : resources) {
            callbacks_.onConfigUpdate(resource);
        }
    }
};
```

---

## 6. Ejemplo: LDS Response

### 6.1 Protobuf Definition

```protobuf
// api/envoy/config/listener/v3/listener.proto

message Listener {
    string name = 1;
    core.v3.Address address = 2;
    repeated FilterChain filter_chains = 3;
    // ...
}
```

### 6.2 Response de Istio

```json
{
  "versionInfo": "2023-12-01T10:30:00Z/123",
  "resources": [
    {
      "@type": "type.googleapis.com/envoy.config.listener.v3.Listener",
      "name": "0.0.0.0_8080",
      "address": {
        "socketAddress": {
          "address": "0.0.0.0",
          "portValue": 8080
        }
      },
      "filterChains": [
        {
          "filters": [
            {
              "name": "envoy.filters.network.http_connection_manager",
              "typedConfig": {
                "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                "rds": {
                  "configSource": {
                    "ads": {}
                  },
                  "routeConfigName": "8080"
                }
              }
            }
          ]
        }
      ]
    }
  ],
  "typeUrl": "type.googleapis.com/envoy.config.listener.v3.Listener",
  "nonce": "abc123"
}
```

---

## 7. xDS en Istio

### 7.1 Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                         istiod                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────┐    ┌────────────────┐    ┌───────────────┐ │
│  │ Config Store   │    │ Service        │    │ Certificate   │ │
│  │ (K8s API)      │    │ Discovery      │    │ Authority     │ │
│  └───────┬────────┘    └───────┬────────┘    └───────┬───────┘ │
│          │                     │                     │          │
│          └─────────────────────┼─────────────────────┘          │
│                                │                                │
│                       ┌────────┴────────┐                       │
│                       │    xDS Server   │                       │
│                       │   (port 15010)  │                       │
│                       └────────┬────────┘                       │
│                                │                                │
└────────────────────────────────┼────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        ┌──────────┐      ┌──────────┐      ┌──────────┐
        │  Envoy   │      │  Envoy   │      │ ztunnel  │
        │ (sidecar)│      │(waypoint)│      │  (node)  │
        └──────────┘      └──────────┘      └──────────┘
```

### 7.2 Lo que Istio Envía

| Recurso | Contenido                           |
| ------- | ----------------------------------- |
| **LDS** | Listeners para puertos de servicios |
| **RDS** | Rutas basadas en VirtualServices    |
| **CDS** | Clusters para cada servicio K8s     |
| **EDS** | Pods endpoints por servicio         |
| **SDS** | Certificados mTLS (SPIFFE)          |

### 7.3 Ejemplo: Service → xDS

```yaml
# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: my-api
  namespace: default
spec:
  ports:
    - port: 8080
  selector:
    app: my-api
---
# Pods
# Pod 1: 10.0.1.5
# Pod 2: 10.0.1.6

# Istio genera:
# CDS: cluster "outbound|8080||my-api.default.svc.cluster.local"
# EDS: endpoints [10.0.1.5:8080, 10.0.1.6:8080]
```

---

## 8. Debugging xDS

### 8.1 Ver Configuración Actual

```bash
# Config dump completo
curl localhost:15000/config_dump

# Solo clusters
curl localhost:15000/config_dump?resource=clusters

# Solo listeners
curl localhost:15000/config_dump?resource=listeners

# Solo routes
curl localhost:15000/config_dump?resource=routes
```

### 8.2 Ver Estado de Sincronización

```bash
# Con istioctl
istioctl proxy-status

# Output:
# NAME               CDS    LDS    EDS    RDS    ISTIOD
# pod-abc.default    SYNCED SYNCED SYNCED SYNCED istiod-1234

# Detalle de config
istioctl proxy-config cluster <pod-name>
istioctl proxy-config listener <pod-name>
istioctl proxy-config route <pod-name>
istioctl proxy-config endpoint <pod-name>
```

### 8.3 Logs de xDS

```bash
# En Envoy, habilitar logging de xDS
--component-log-level upstream:debug,config:debug
```

---

## 9. Delta xDS

### 9.1 State of the World vs Delta

```
State of the World (SotW):
  Cada response contiene TODOS los recursos
  - Simple pero ineficiente para updates pequeños

Delta xDS:
  Solo envía cambios (added, removed, updated)
  - Más eficiente para clusters grandes
```

### 9.2 Delta Request

```protobuf
message DeltaDiscoveryRequest {
    Node node = 1;
    string type_url = 2;
    repeated string resource_names_subscribe = 3;    // Nuevos recursos
    repeated string resource_names_unsubscribe = 4;  // Recursos a quitar
    map<string, string> initial_resource_versions = 5;
}

message DeltaDiscoveryResponse {
    repeated Resource resources = 1;  // Recursos nuevos/actualizados
    repeated string removed_resources = 2;  // Recursos eliminados
    string nonce = 3;
}
```

---

## 10. Autoevaluación

1. ¿Qué problema resuelve xDS?
2. ¿Cuál es la diferencia entre ACK y NACK?
3. ¿Por qué existe ADS?
4. ¿En qué orden se entregan los recursos con ADS?
5. ¿Cómo puedes ver la configuración xDS de un Envoy?

---

**Siguiente Módulo**: [../04_ztunnel_arquitectura/01_ambient_mode_context.md](../04_ztunnel_arquitectura/01_ambient_mode_context.md) - Arquitectura de ztunnel
