# Plan de Formación: Proxies de Red, Envoy y ztunnel

---

## 1. Contexto y Objetivos

### 1.1 Perfil del Usuario

| Aspecto | Detalle |
|---------|---------|
| **Experiencia general** | 20+ años en desarrollo de software |
| **Lenguajes principales** | C++ y Java |
| **Conocimientos de Rust/Go** | Básicos, no son lenguajes principales |
| **Conocimientos de redes/proxies** | Limitados, recién iniciado en este dominio |
| **Situación actual** | Recién incorporado a proyectos Envoy y ztunnel en Red Hat |

### 1.2 Objetivos de Aprendizaje

El objetivo final es **entender perfectamente cómo funciona un proxy de red** y cómo implementan estas funcionalidades Envoy y ztunnel. Específicamente:

1. **Fundamentos conceptuales**:
   - ¿Cuál es el objetivo de un proxy de red?
   - ¿Es el mismo objetivo a L4 que a L7?
   - ¿Qué diferencias hay entre proxies L4 y L7?
   - ¿Qué ventajas tiene uno sobre el otro?

2. **Visión técnica clara**:
   - Cómo funciona un proxy de red a L7
   - Cómo funciona un proxy de red a L4
   - Manejo de distintos protocolos y tipos de paquetes
   - Flujo de datos desde la conexión hasta el upstream

3. **Aplicación práctica**:
   - Cómo aplican estos conceptos a Envoy
   - Cómo aplican estos conceptos a ztunnel
   - Cuándo usar cada uno
   - Cómo trabajan juntos en Istio ambient mode

4. **Base para el futuro**:
   - Cimentar conocimientos para investigaciones posteriores
   - Capacidad de contribuir al código de ambos proyectos
   - Debugging efectivo de problemas en producción

### 1.3 Repositorios de Trabajo

| Proyecto | Ubicación | Lenguaje |
|----------|-----------|----------|
| **Envoy** | `/home/jojosneg/source/redhat/envoy/upstream/main` | C++20 |
| **ztunnel** | `/home/jojosneg/source/redhat/ztunnel/upstream/master` | Rust |

---

## 2. Directrices de Creación de Documentos

### 2.1 Preferencias Confirmadas por el Usuario

| Aspecto | Preferencia |
|---------|-------------|
| **Formato** | Documentos Markdown (.md) |
| **Ubicación** | Directorio separado: `~/source/redhat/proxy-training/` |
| **Profundidad** | **Deep dive** con análisis exhaustivo de código fuente |
| **Idioma** | **Mixto**: conceptos en español, términos técnicos en inglés |

### 2.2 Estructura de Cada Documento

Cada documento DEBE incluir:

```markdown
# Título del Tema

---
**Módulo**: X - Nombre del Módulo
**Tema**: Nombre del tema
**Tiempo estimado**: X horas
**Prerrequisitos**: [lista de documentos previos]
---

## Objetivos de Aprendizaje

Al completar este documento:
- Objetivo 1
- Objetivo 2
- Objetivo 3

---

## 1. Contenido Principal

### 1.1 Sección con teoría

Explicación conceptual en español.
Términos técnicos en inglés (con definición la primera vez que aparecen).

### 1.2 Diagramas

Usar diagramas ASCII o Mermaid para visualizar:
- Arquitecturas
- Flujos de datos
- Secuencias de operaciones

### 1.3 Deep Dive de Código

Referencias exactas al código fuente:
- Formato: `archivo.cc:línea` o `archivo.cc:línea-línea`
- Ejemplo: `source/common/http/conn_manager_impl.cc:394-420`

Incluir fragmentos de código relevantes con explicación línea por línea
de funciones críticas.

Explicar:
- Decisiones de diseño
- Patrones arquitectónicos identificados
- Por qué se implementó de esa manera

---

## N. Ejercicios Prácticos

- Comandos exactos para ejecutar
- Checkpoints para verificar progreso
- Preguntas de reflexión

---

## N+1. Autoevaluación

1. Pregunta conceptual 1
2. Pregunta conceptual 2
3. Pregunta práctica 1
4. etc.

---

## Referencias en el Código

| Archivo | Descripción |
|---------|-------------|
| `path/to/file.cc:line` | Qué hace |

---

**Siguiente**: [link al siguiente documento]
```

### 2.3 Principios de Contenido

1. **Pedagogía progresiva**:
   - Construir conocimiento de forma incremental
   - Cada módulo se basa en los anteriores
   - No asumir conocimientos no cubiertos previamente

2. **Enfoque práctico**:
   - Teoría siempre acompañada de ejemplos concretos
   - Referencias constantes al código real
   - Ejercicios que refuercen el aprendizaje

3. **Deep dive técnico**:
   - No superficial: análisis exhaustivo
   - Explicar el "por qué", no solo el "qué"
   - Incluir decisiones de diseño y trade-offs

4. **Idioma mixto**:
   - Narrativa y explicaciones: **español**
   - Términos técnicos: **inglés** (la primera vez con definición)
   - Código y comandos: **inglés** (como en el código fuente)
   - Nombres de archivos y paths: tal cual están

5. **Referencias precisas**:
   - Siempre indicar archivo y línea: `file.cc:123`
   - Permitir navegación directa al código
   - Actualizar si el código cambia

### 2.4 Estilo de Diagramas

Preferir diagramas ASCII para compatibilidad:

```
┌─────────────────┐
│   Componente    │
├─────────────────┤
│ - Propiedad 1   │
│ - Propiedad 2   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Otro componente│
└─────────────────┘
```

Para flujos complejos, usar Mermaid cuando sea necesario.

### 2.5 Nivel de Detalle del Código

Para análisis de código, incluir:

```cpp
// source/common/http/conn_manager_impl.cc:394-420

// EXPLICACIÓN: Esta función se llama cuando llegan headers HTTP
void ConnectionManagerImpl::onHeadersComplete(StreamDecoderPtr&& decoder) {
    // Crear nuevo ActiveStream para este request
    // ActiveStream mantiene el estado del request durante su vida
    auto stream = std::make_unique<ActiveStream>(*this, decoder);

    // Empezar el procesamiento de la filter chain
    // decodeHeaders() ejecuta cada filter en orden
    stream->decodeHeaders(std::move(headers), end_stream);
}

// POR QUÉ SE DISEÑÓ ASÍ:
// - ActiveStream encapsula todo el estado del request
// - Permite múltiples requests concurrentes (HTTP/2)
// - La filter chain es extensible sin modificar este código
```

---

## 3. Estructura del Plan de Formación

### 3.1 Módulos

El plan se organiza en **6 módulos progresivos**:

| Módulo | Nombre | Objetivo | Duración Est. |
|--------|--------|----------|---------------|
| 1 | Fundamentos de Redes | Base teórica: OSI, TCP, HTTP, TLS | 1-2 días |
| 2 | Proxies - Conceptos | Qué es un proxy, L4 vs L7, casos de uso | 2-3 días |
| 3 | Arquitectura de Envoy | Deep dive en Envoy: threading, filters, xDS | 4-5 días |
| 4 | Arquitectura de ztunnel | Deep dive en ztunnel: Tokio, HBONE, ambient | 3-4 días |
| 5 | Comparativa | Envoy vs ztunnel, cuándo usar cada uno | 2 días |
| 6 | Avanzado | Temas especializados según necesidad | Ongoing |

### 3.2 Estructura de Directorios

```
~/source/redhat/proxy-training/
│
├── README.md                         # Índice y guía de uso
├── PLAN_FORMACION.md                 # Este documento
│
├── 01_fundamentos_redes/
│   ├── 01_modelo_osi.md              # Modelo OSI, capas L3/L4/L7
│   ├── 02_tcp_udp.md                 # TCP handshake, UDP, sockets
│   ├── 03_http_protocols.md          # HTTP/1.1, HTTP/2, gRPC
│   └── 04_tls_mtls.md                # TLS, mTLS, SPIFFE, certs
│
├── 02_proxies_conceptos/
│   ├── 01_que_es_proxy.md            # Tipos, forward/reverse
│   ├── 02_proxy_l4.md                # Proxy L4 deep dive
│   ├── 03_proxy_l7.md                # Proxy L7 deep dive
│   └── 04_comparativa.md             # L4 vs L7 decisión
│
├── 03_envoy_arquitectura/
│   ├── 01_vision_general.md          # Arquitectura, componentes
│   ├── 02_threading_model.md         # Threading, event loop, TLS
│   ├── 03_life_of_request.md         # Flujo completo de request
│   ├── 04_filter_chains.md           # Network y HTTP filters
│   ├── 05_cluster_management.md      # Clusters, LB, health checks
│   ├── 06_xds_protocol.md            # xDS, ADS, config dinámica
│   └── code_analysis/                # Deep dives opcionales
│       ├── conn_manager_deep_dive.md
│       ├── router_filter_analysis.md
│       └── load_balancer_internals.md
│
├── 04_ztunnel_arquitectura/
│   ├── 01_ambient_mode_context.md    # Istio ambient, por qué ztunnel
│   ├── 02_threading_tokio.md         # Tokio runtime, async/await
│   ├── 03_hbone_protocol.md          # HBONE, HTTP CONNECT, mTLS
│   ├── 04_traffic_redirection.md     # iptables, redirección
│   └── code_analysis/                # Deep dives opcionales
│       ├── proxy_module.md
│       ├── xds_integration.md
│       └── identity_management.md
│
├── 05_comparativa/
│   ├── 01_envoy_vs_ztunnel.md        # Comparación detallada
│   ├── 02_ambient_mode_completo.md   # Arquitectura completa
│   └── 03_decision_framework.md      # Marco de decisión
│
├── 06_avanzado/
│   ├── envoy/
│   │   ├── custom_filters.md         # Escribir filtros
│   │   ├── hot_restart.md            # Hot restart
│   │   └── performance_tuning.md     # Tuning
│   └── ztunnel/
│       ├── rust_async_patterns.md    # Async en Rust
│       └── certificate_rotation.md   # Rotación de certs
│
├── ejercicios/
│   ├── envoy/
│   │   ├── 01_compilar_ejecutar.md
│   │   ├── 02_trazar_request.md
│   │   └── 03_modificar_filtro.md
│   └── ztunnel/
│       ├── 01_build_run.md
│       └── 02_analizar_conexion.md
│
└── recursos/
    ├── referencias.md                # Links y documentación
    ├── diagramas/                    # Diagramas reutilizables
    └── config_examples/              # Ejemplos de configuración
```

---

## 4. Estado de Generación de Documentos

**Última actualización**: 2025-12-04

### 4.1 Resumen de Progreso

| Módulo | Documentos | Estado | % |
|--------|------------|--------|---|
| Módulo 1: Fundamentos Redes | 4/4 | ✅ Completo | 100% |
| Módulo 2: Proxies Conceptos | 4/4 | ✅ Completo | 100% |
| Módulo 3: Envoy Arquitectura | 6/6 | ✅ Completo | 100% |
| Módulo 4: ztunnel Arquitectura | 4/4 | ✅ Completo | 100% |
| Módulo 5: Comparativa | 3/3 | ✅ Completo | 100% |
| Módulo 6: Avanzado | 5/5 | ✅ Completo | 100% |
| Ejercicios | 5/5 | ✅ Completo | 100% |
| Recursos | 2/2 | ✅ Completo | 100% |

**Total**: 33/33 documentos (100% completado)

### 4.2 Detalle por Documento

#### ✅ Módulo 1: Fundamentos de Redes (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `01_modelo_osi.md` | Modelo OSI, capas 1-7, foco en L3/L4/L7, encapsulación | ✅ |
| `02_tcp_udp.md` | TCP handshake, UDP, sockets, connection pooling | ✅ |
| `03_http_protocols.md` | HTTP/1.1, HTTP/2, gRPC, multiplexing, HPACK | ✅ |
| `04_tls_mtls.md` | TLS 1.2/1.3, mTLS, SPIFFE, X.509, certificados | ✅ |

#### ✅ Módulo 2: Proxies Conceptos (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `01_que_es_proxy.md` | Forward/reverse proxy, transparent proxy, service mesh | ✅ |
| `02_proxy_l4.md` | L4 deep dive, qué ve/no ve, ztunnel como ejemplo | ✅ |
| `03_proxy_l7.md` | L7 deep dive, filter chain, HTTP parsing, Envoy | ✅ |
| `04_comparativa.md` | L4 vs L7, framework de decisión, casos de uso | ✅ |

#### ✅ Módulo 3: Envoy Arquitectura (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `01_vision_general.md` | Arquitectura, data/control plane, estructura código | ✅ |
| `02_threading_model.md` | Main/worker threads, event loop, TLS, hot restart | ✅ |
| `03_life_of_request.md` | Flujo completo: listener → filters → router → upstream | ✅ |
| `04_filter_chains.md` | Network filters, HTTP filters, interfaces, ejemplos | ✅ |
| `05_cluster_management.md` | Clusters, LB algorithms, health checks, circuit breaking | ✅ |
| `06_xds_protocol.md` | LDS/CDS/RDS/EDS/SDS, ADS, ACK/NACK, Istio integration | ✅ |

#### ✅ Módulo 4: ztunnel Arquitectura (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `01_ambient_mode_context.md` | Por qué ambient, ztunnel vs sidecar, arquitectura | ✅ |
| `02_threading_tokio.md` | Tokio runtime, async/await, comparación con Envoy | ✅ |
| `03_hbone_protocol.md` | HBONE, HTTP/2 CONNECT, mTLS, SPIFFE certs | ✅ |
| `04_traffic_redirection.md` | iptables, in-pod redirection, istio-cni | ✅ |

#### ✅ Módulo 5: Comparativa (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `01_envoy_vs_ztunnel.md` | Tabla comparativa, casos de uso, trade-offs | ✅ |
| `02_ambient_mode_completo.md` | ztunnel + waypoint, flujo completo | ✅ |
| `03_decision_framework.md` | Árbol de decisión, recomendaciones | ✅ |

#### ✅ Módulo 6: Avanzado (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `envoy/custom_filters.md` | Cómo escribir un HTTP filter | ✅ |
| `envoy/hot_restart.md` | Mecanismo de hot restart | ✅ |
| `envoy/performance_tuning.md` | pprof, stats, optimización | ✅ |
| `ztunnel/rust_async_patterns.md` | Patrones async en Rust | ✅ |
| `ztunnel/certificate_rotation.md` | Rotación de certs SPIFFE | ✅ |

#### ✅ Ejercicios (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `envoy/01_compilar_ejecutar.md` | Build y run de Envoy | ✅ |
| `envoy/02_trazar_request.md` | Añadir logging, trazar request | ✅ |
| `envoy/03_modificar_filtro.md` | Modificar un filter existente | ✅ |
| `ztunnel/01_build_run.md` | Build y run de ztunnel | ✅ |
| `ztunnel/02_analizar_conexion.md` | Analizar flujo de conexión | ✅ |

#### ✅ Recursos (COMPLETO)

| Archivo | Contenido | Estado |
|---------|-----------|--------|
| `README.md` | Índice completo, guía de uso | ✅ |
| `recursos/referencias.md` | Links oficiales, RFCs, herramientas | ✅ |

---

## 5. Archivos Clave para Análisis de Código

### 5.1 Envoy

| Archivo | Propósito | Prioridad |
|---------|-----------|-----------|
| `source/common/http/conn_manager_impl.h:60` | HTTP Connection Manager core | Alta |
| `source/extensions/filters/http/router/router.cc:400-800` | Router filter, forwarding | Alta |
| `source/common/upstream/cluster_manager_impl.h:98` | Cluster management | Alta |
| `source/common/upstream/load_balancer_impl.cc:200-400` | LB algorithms | Media |
| `source/common/event/dispatcher_impl.h:36` | Event loop | Media |
| `source/server/worker_impl.h:30-100` | Worker threads | Media |
| `source/common/network/filter_manager_impl.cc:100-300` | Filter chain execution | Media |
| `source/common/config/grpc_mux_impl.cc` | xDS subscription | Media |

### 5.2 ztunnel

| Archivo | Propósito | Prioridad |
|---------|-----------|-----------|
| `ARCHITECTURE.md` | Threading, puertos, diseño | Alta |
| `README.md` | Overview, métricas, TLS providers | Alta |
| `src/proxy/` | Core proxy implementation | Alta |
| `src/xds/` | xDS client | Media |
| `src/identity/` | SPIFFE certificate management | Media |

---

## 6. Plan de Estudio Recomendado

### Semana 1: Fundamentos
- Completar Módulos 1 y 2
- Leer documentación teórica
- Familiarizarse con terminología

### Semanas 2-3: Envoy
- Completar Módulo 3
- Compilar y ejecutar Envoy
- Trazar requests, analizar código
- Ejercicios prácticos

### Semanas 3-4: ztunnel
- Completar Módulo 4
- Familiarizarse con Rust async
- Compilar y analizar ztunnel
- Ejercicios prácticos

### Semana 5: Integración
- Completar Módulo 5
- Entender ambient mode completo
- Experimentar con ambos proyectos

### Ongoing: Especialización
- Módulo 6 según necesidades del trabajo
- Contribuciones al código
- Casos de uso específicos

---

## 7. Recursos de Referencia

### Documentación Oficial
- [Envoy Docs](https://www.envoyproxy.io/docs/envoy/latest/)
- [Istio Docs](https://istio.io/latest/docs/)
- [Istio Ambient Architecture](https://istio.io/latest/docs/ambient/architecture/)

### Artículos Clave
- [Envoy Threading Model](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)
- [Life of a Request](https://www.envoyproxy.io/docs/envoy/latest/intro/life_of_a_request)
- [Rust-Based Ztunnel](https://istio.io/latest/blog/2023/rust-based-ztunnel/)

### En los Repositorios
- Envoy: `docs/analysis/00_architecture_overview.md`
- ztunnel: `ARCHITECTURE.md`, `README.md`

---

## 8. Cómo Continuar la Generación

Para continuar generando documentos pendientes:

```
Continúa generando los documentos pendientes del plan de formación
```

Para un documento específico:

```
Genera el documento 04_traffic_redirection.md del Módulo 4
```

Para regenerar un documento:

```
Regenera el documento X con más detalle en la sección Y
```

---

*Plan creado: 2025-12-04*
*Última actualización: 2025-12-04*
