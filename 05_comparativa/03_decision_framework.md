# Framework de Decisión: Eligiendo el Proxy Correcto

---

**Módulo**: 5 - Comparativa
**Tema**: Guía de decisión para arquitecturas de proxy
**Tiempo estimado**: 1.5 horas
**Prerrequisitos**: [02_ambient_mode_completo.md](02_ambient_mode_completo.md)

---

## Objetivos de Aprendizaje

Al completar este documento:

- Tendrás un framework claro para elegir tecnologías
- Conocerás los criterios de decisión clave
- Podrás justificar decisiones arquitectónicas
- Sabrás cuándo usar cada combinación de componentes

---

## 1. Árbol de Decisión Principal

### 1.1 Decisión de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────┐
│                    DECISION TREE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ¿Necesitas Service Mesh?                                      │
│  │                                                              │
│  ├── NO ────────────────────────────────────────────────────── │
│  │   │                                                         │
│  │   └── ¿Qué necesitas?                                      │
│  │       │                                                     │
│  │       ├── Solo load balancing ───────> Cloud LB / HAProxy  │
│  │       ├── Solo ingress ──────────────> Nginx / Traefik     │
│  │       └── API Gateway ───────────────> Kong / Envoy Gateway│
│  │                                                             │
│  └── SÍ ───────────────────────────────────────────────────── │
│      │                                                         │
│      └── ¿Qué % de servicios necesitan L7?                    │
│          │                                                     │
│          ├── Mayoría (>70%) ────────────> Sidecar Mode (Envoy)│
│          │                                                     │
│          ├── Minoría (<30%) ────────────> Ambient + Waypoints │
│          │                                                     │
│          └── Mixto (30-70%)                                   │
│              │                                                 │
│              └── ¿Recursos son críticos?                      │
│                  │                                             │
│                  ├── SÍ ────────────────> Ambient + Waypoints │
│                  └── NO ────────────────> Sidecar Mode        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Decisión de Modo de Istio

```
┌─────────────────────────────────────────────────────────────────┐
│              SIDECAR vs AMBIENT DECISION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    SIDECAR MODE                          │   │
│  │                                                          │   │
│  │  Elegir cuando:                                          │   │
│  │  ✓ Necesitas L7 para casi todos los servicios           │   │
│  │  ✓ Usas extensivamente WASM/Lua                         │   │
│  │  ✓ Compliance requiere proxy por pod                    │   │
│  │  ✓ Ya tienes inversión en sidecar config               │   │
│  │  ✓ Necesitas máximo aislamiento de fallas              │   │
│  │                                                          │   │
│  │  Trade-offs:                                             │   │
│  │  • +50-100MB RAM por pod                                │   │
│  │  • Restart de pods para upgrades                        │   │
│  │  • Mayor latencia (+1-5ms)                              │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   AMBIENT MODE                           │   │
│  │                                                          │   │
│  │  Elegir cuando:                                          │   │
│  │  ✓ Mayoría de servicios solo necesitan mTLS             │   │
│  │  ✓ Recursos son limitados (muchos pods pequeños)        │   │
│  │  ✓ Upgrades frecuentes del mesh                         │   │
│  │  ✓ Latencia L4 es crítica                               │   │
│  │  ✓ Simplicidad operativa es prioridad                   │   │
│  │                                                          │   │
│  │  Trade-offs:                                             │   │
│  │  • L7 requiere waypoint adicional                       │   │
│  │  • Menos battle-tested                                  │   │
│  │  • Algunas features no disponibles                      │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Criterios de Evaluación

### 2.1 Matriz de Requisitos

```
┌─────────────────────────────────────────────────────────────────┐
│                   REQUIREMENTS MATRIX                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Puntúa cada criterio de 1-5 según tu caso:                    │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Criterio                              │ Peso │ Tu Score   │ │
│  ├───────────────────────────────────────┼──────┼────────────┤ │
│  │ Necesidad de features L7              │  3   │ ___        │ │
│  │ Criticidad de uso de recursos         │  2   │ ___        │ │
│  │ Frecuencia de upgrades mesh           │  2   │ ___        │ │
│  │ Requisitos de latencia                │  2   │ ___        │ │
│  │ Necesidad de extensibilidad           │  2   │ ___        │ │
│  │ Madurez/estabilidad requerida         │  2   │ ___        │ │
│  │ Complejidad operativa tolerable       │  1   │ ___        │ │
│  └───────────────────────────────────────┴──────┴────────────┘ │
│                                                                 │
│  Scoring Guide:                                                │
│  1 = No importante / No necesito                               │
│  5 = Muy importante / Necesito mucho                           │
│                                                                 │
│  Interpretación:                                               │
│  • L7 alto + extensibilidad alta → Sidecar                    │
│  • Recursos + latencia alto → Ambient                          │
│  • Madurez alta → Sidecar (más probado)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Checklist de Features

#### Para L7 (¿Necesito Envoy/Waypoint?)

| Feature                     | ¿La necesito? | Notas       |
| --------------------------- | ------------- | ----------- |
| HTTP routing por path       | □ Sí □ No     | Requiere L7 |
| HTTP routing por headers    | □ Sí □ No     | Requiere L7 |
| JWT validation              | □ Sí □ No     | Requiere L7 |
| Rate limiting L7            | □ Sí □ No     | Requiere L7 |
| Request/Response transforms | □ Sí □ No     | Requiere L7 |
| gRPC transcoding            | □ Sí □ No     | Requiere L7 |
| WASM extensions             | □ Sí □ No     | Solo Envoy  |
| Circuit breaking L7         | □ Sí □ No     | Requiere L7 |
| Retries inteligentes        | □ Sí □ No     | Requiere L7 |
| Mirroring                   | □ Sí □ No     | Requiere L7 |

**Si >5 checks en Sí**: Necesitas L7 extensivo → Sidecar o Ambient+Waypoint

#### Para L4 (¿ztunnel es suficiente?)

| Feature               | ¿La necesito? | ztunnel? |
| --------------------- | ------------- | -------- |
| mTLS automático       | □ Sí □ No     | ✓ Sí     |
| Identidad SPIFFE      | □ Sí □ No     | ✓ Sí     |
| Authorization L4      | □ Sí □ No     | ✓ Sí     |
| Métricas TCP          | □ Sí □ No     | ✓ Sí     |
| Cifrado en tránsito   | □ Sí □ No     | ✓ Sí     |
| Zero trust networking | □ Sí □ No     | ✓ Sí     |

**Si todo es Sí y L7 es No**: ztunnel (ambient sin waypoint) es suficiente

---

## 3. Escenarios Comunes

### 3.1 Microservicios Internos

```
┌─────────────────────────────────────────────────────────────────┐
│  Escenario: Backend microservices (no user-facing)             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Características típicas:                                      │
│  • Comunicación service-to-service                             │
│  • No JWT (tokens internos o ninguno)                          │
│  • Routing simple (service name → destino)                     │
│  • Requisito principal: mTLS, observabilidad                   │
│                                                                 │
│  RECOMENDACIÓN: Ambient (solo ztunnel)                         │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  [Service A] ──> ztunnel ══> ztunnel ──> [Service B]    │   │
│  │                      mTLS tunnel                         │   │
│  │                                                          │   │
│  │  Beneficios:                                             │   │
│  │  • Mínimo overhead                                       │   │
│  │  • Sin parsing HTTP innecesario                         │   │
│  │  • Máxima eficiencia                                    │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 API Pública

```
┌─────────────────────────────────────────────────────────────────┐
│  Escenario: Public API (user-facing)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Características típicas:                                      │
│  • Clientes externos                                           │
│  • JWT/OAuth validation                                        │
│  • Rate limiting                                               │
│  • Routing por path/version                                    │
│  • Políticas de seguridad complejas                           │
│                                                                 │
│  RECOMENDACIÓN: Ambient + Waypoint para API services           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  [Client] ──> Ingress ──> ztunnel ──> Waypoint          │   │
│  │                                          │               │   │
│  │                   JWT validation ◄───────┘               │   │
│  │                   Rate limiting                          │   │
│  │                   L7 routing                             │   │
│  │                          │                               │   │
│  │                          ▼                               │   │
│  │              ztunnel ──> [API Service]                   │   │
│  │                                                          │   │
│  │  Beneficios:                                             │   │
│  │  • L7 solo donde necesario                              │   │
│  │  • Backend services sin overhead L7                     │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Legacy Migration

```
┌─────────────────────────────────────────────────────────────────┐
│  Escenario: Migrating from monolith                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Características típicas:                                      │
│  • Mix de servicios nuevos y legacy                            │
│  • Adoption gradual                                            │
│  • Compatibilidad hacia atrás requerida                        │
│  • Algunos servicios no pueden tener sidecar                   │
│                                                                 │
│  RECOMENDACIÓN: Ambient (adopción gradual)                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Fase 1: Solo mTLS                                      │   │
│  │  [Legacy] ──> ztunnel ══> ztunnel ──> [New Service]     │   │
│  │                                                          │   │
│  │  Fase 2: Añadir policies                                │   │
│  │  + AuthorizationPolicy L4                               │   │
│  │                                                          │   │
│  │  Fase 3: L7 donde necesario                             │   │
│  │  + Waypoint para servicios específicos                  │   │
│  │                                                          │   │
│  │  Beneficios:                                             │   │
│  │  • No requiere modificar pods legacy                    │   │
│  │  • Adopción incremental                                 │   │
│  │  • Rollback fácil                                       │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 High-Performance Trading

```
┌─────────────────────────────────────────────────────────────────┐
│  Escenario: Ultra-low latency (fintech, gaming)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Características típicas:                                      │
│  • Cada microsegundo cuenta                                    │
│  • Alto throughput                                             │
│  • Mínimo overhead                                             │
│  • Security sigue siendo requerido                             │
│                                                                 │
│  RECOMENDACIÓN: Ambient (solo L4) o bypass selectivo           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Opción A: Ambient sin waypoint                         │   │
│  │  [Trading Engine] ──> ztunnel ──> [Market Data]         │   │
│  │                          │                               │   │
│  │                    ~0.1-0.5ms                           │   │
│  │                                                          │   │
│  │  Opción B: Bypass para paths críticos                   │   │
│  │  • Tráfico ultra-crítico: direct (sin mesh)             │   │
│  │  • Resto: ambient mesh                                  │   │
│  │                                                          │   │
│  │  Consideración:                                          │   │
│  │  • Evaluar si mTLS overhead es aceptable                │   │
│  │  • TLS handshake es el mayor costo                      │   │
│  │  • Connection pooling mitiga esto                       │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Calculadora de Recursos

### 4.1 Estimación de RAM

```
┌─────────────────────────────────────────────────────────────────┐
│                   RESOURCE CALCULATOR                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Inputs:                                                       │
│  • Número de pods: ___                                         │
│  • Número de nodos: ___                                        │
│  • Pods con requisitos L7: ___                                 │
│                                                                 │
│  SIDECAR MODE:                                                 │
│  ───────────────────────────────────────────                    │
│  RAM total = pods × 75MB (promedio)                            │
│                                                                 │
│  Ejemplo: 500 pods                                             │
│  RAM = 500 × 75MB = 37.5 GB                                    │
│                                                                 │
│  AMBIENT MODE (sin waypoint):                                  │
│  ───────────────────────────────────────────                    │
│  RAM total = nodos × 50MB                                      │
│                                                                 │
│  Ejemplo: 20 nodos, 500 pods                                   │
│  RAM = 20 × 50MB = 1 GB                                        │
│                                                                 │
│  AMBIENT MODE (con waypoints):                                 │
│  ───────────────────────────────────────────                    │
│  RAM total = (nodos × 50MB) + (waypoints × 100MB)              │
│                                                                 │
│  Ejemplo: 20 nodos, 5 waypoints                                │
│  RAM = (20 × 50MB) + (5 × 100MB) = 1.5 GB                      │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  AHORRO: 37.5GB - 1.5GB = 36GB (96% menos!)           │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Estimación de Latencia

```
┌─────────────────────────────────────────────────────────────────┐
│                   LATENCY COMPARISON                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Baseline (sin proxy):                 ~0ms added              │
│                                                                 │
│  ztunnel only (L4):                                            │
│  ┌────────────────────────────────────────────────┐            │
│  │ TCP proxy overhead:     ~0.05ms               │            │
│  │ mTLS handshake:         ~0.1-0.3ms (first)    │            │
│  │ mTLS reuse:             ~0ms                  │            │
│  │ HBONE overhead:         ~0.05ms               │            │
│  │ ─────────────────────────────────             │            │
│  │ Total per-connection:   ~0.1-0.5ms            │            │
│  │ Total per-request:      ~0.1ms (connection reuse)          │
│  └────────────────────────────────────────────────┘            │
│                                                                 │
│  Envoy sidecar (L7):                                           │
│  ┌────────────────────────────────────────────────┐            │
│  │ HTTP parsing:           ~0.1-0.5ms            │            │
│  │ Filter chain:           ~0.1-1.0ms            │            │
│  │ mTLS:                   ~0.1-0.3ms            │            │
│  │ ─────────────────────────────────             │            │
│  │ Total per-request:      ~1-5ms                │            │
│  └────────────────────────────────────────────────┘            │
│                                                                 │
│  Ambient + Waypoint:                                           │
│  ┌────────────────────────────────────────────────┐            │
│  │ ztunnel hop 1:          ~0.1ms                │            │
│  │ Waypoint (L7):          ~1-2ms                │            │
│  │ ztunnel hop 2:          ~0.1ms                │            │
│  │ ─────────────────────────────────             │            │
│  │ Total per-request:      ~1.5-2.5ms            │            │
│  └────────────────────────────────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Decision Matrix Final

### 5.1 Tabla de Decisión Rápida

| Situación                         | Recomendación                  | Razón                              |
| --------------------------------- | ------------------------------ | ---------------------------------- |
| Cluster nuevo, requisitos simples | Ambient                        | Menos overhead, más simple         |
| Cluster existente con sidecars    | Mantener sidecar               | Migración tiene costo              |
| Todos los services necesitan L7   | Sidecar                        | Evita hop adicional de waypoint    |
| <20% services necesitan L7        | Ambient + waypoints selectivos | Mejor uso de recursos              |
| Ultra-low latency requerido       | Ambient (L4) o bypass          | Mínimo overhead                    |
| WASM extensivo                    | Sidecar                        | Ambient no soporta WASM en ztunnel |
| Compliance estricto               | Sidecar                        | Más maduro, mejor documentado      |
| Recursos muy limitados            | Ambient                        | 10-50x menos RAM                   |
| Upgrades frecuentes del mesh      | Ambient                        | No requiere restart pods           |

### 5.2 Flowchart de Decisión

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUICK DECISION FLOWCHART                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  START                                                         │
│    │                                                            │
│    ▼                                                            │
│  ┌─────────────────────────────────┐                           │
│  │ ¿Cluster nuevo o migración?     │                           │
│  └────────────┬────────────────────┘                           │
│               │                                                 │
│    ┌──────────┴──────────┐                                     │
│    │                     │                                      │
│    ▼ Nuevo               ▼ Migración                           │
│    │                     │                                      │
│    │              ┌──────┴──────┐                              │
│    │              │ ¿Funciona   │                              │
│    │              │ actualmente?│                              │
│    │              └──────┬──────┘                              │
│    │                     │                                      │
│    │          ┌──────────┴──────────┐                          │
│    │          │ Sí                  │ No                       │
│    │          ▼                     ▼                          │
│    │    Mantener             Evaluar Ambient                   │
│    │    Sidecar                    │                           │
│    │                               │                           │
│    ▼                               ▼                           │
│  ┌─────────────────────────────────────────┐                   │
│  │ ¿Mayoría de services necesitan L7?      │                   │
│  └────────────────────┬────────────────────┘                   │
│                       │                                         │
│         ┌─────────────┴─────────────┐                          │
│         │ Sí (>70%)                 │ No (<70%)                │
│         ▼                           ▼                          │
│    ┌──────────┐              ┌──────────────┐                  │
│    │ SIDECAR  │              │   AMBIENT    │                  │
│    │  MODE    │              └──────┬───────┘                  │
│    └──────────┘                     │                          │
│                                     ▼                          │
│                        ┌───────────────────────┐               │
│                        │ ¿Algunos necesitan L7?│               │
│                        └───────────┬───────────┘               │
│                                    │                           │
│                     ┌──────────────┴──────────────┐            │
│                     │ Sí                          │ No         │
│                     ▼                             ▼            │
│              ┌────────────────┐          ┌─────────────────┐   │
│              │ AMBIENT +      │          │ AMBIENT         │   │
│              │ WAYPOINTS      │          │ (ztunnel only)  │   │
│              └────────────────┘          └─────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Checklist de Implementación

### 6.1 Pre-Implementation

```
□ Evaluar requisitos de cada servicio (L4 vs L7)
□ Inventario de policies existentes
□ Verificar compatibilidad de CNI
□ Estimar recursos necesarios
□ Definir estrategia de rollout
□ Preparar plan de rollback
□ Documentar decisión y justificación
```

### 6.2 Implementation (Ambient)

```
□ Instalar Istio con perfil ambient
□ Verificar ztunnel DaemonSet healthy
□ Verificar istio-cni instalado
□ Habilitar ambient en namespace piloto
□ Validar mTLS funcionando
□ Añadir waypoints donde necesario
□ Configurar AuthorizationPolicies
□ Configurar observabilidad
□ Rollout a más namespaces
```

### 6.3 Post-Implementation

```
□ Validar métricas y logging
□ Verificar latencia aceptable
□ Confirmar policies funcionando
□ Documentar arquitectura final
□ Training del equipo
□ Runbook de operaciones
```

---

## 7. Autoevaluación

1. ¿Cuáles son los 3 factores principales para elegir sidecar vs ambient?
2. ¿Cuándo añadirías un waypoint en ambient mode?
3. ¿Cómo estimarías el ahorro de recursos con ambient?
4. ¿Qué escenario NO es adecuado para ambient mode?
5. ¿Cuál sería tu recomendación para un cluster con 1000 pods donde solo 50 necesitan JWT validation?

---

## 8. Conclusión

### Regla General

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Start with the minimum that solves your problem"             │
│                                                                 │
│  1. ¿Solo necesitas mTLS? → Ambient sin waypoint               │
│  2. ¿Algunos services L7? → Ambient + waypoints selectivos     │
│  3. ¿Todo necesita L7?    → Sidecar mode                       │
│                                                                 │
│  Siempre puedes añadir complejidad después,                    │
│  pero quitarla es más difícil.                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

**Siguiente Módulo**: [../06_avanzado/envoy/custom_filters.md](../06_avanzado/envoy/custom_filters.md) - Temas Avanzados
