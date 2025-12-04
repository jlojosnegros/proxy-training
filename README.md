# Formación: Proxies de Red, Envoy y ztunnel

> **Objetivo**: Dominar los fundamentos de proxies de red y las arquitecturas de Envoy y ztunnel para convertirse en un experto en service mesh y data plane proxies.

---

## Contexto

Este material está diseñado para un desarrollador con:
- 20+ años de experiencia en C++/Java
- Conocimientos básicos de Rust/Go
- Conocimientos limitados de redes y proxies

## Repositorios de Referencia

| Proyecto | Ubicación | Lenguaje |
|----------|-----------|----------|
| Envoy | `/home/jojosneg/source/redhat/envoy/upstream/main` | C++20 |
| ztunnel | `/home/jojosneg/source/redhat/ztunnel/upstream/master` | Rust |

---

## Índice de Contenidos

### Módulo 1: Fundamentos de Redes

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [01_modelo_osi.md](01_fundamentos_redes/01_modelo_osi.md) | Modelo OSI con foco en L3, L4, L7 | 2h |
| [02_tcp_udp.md](01_fundamentos_redes/02_tcp_udp.md) | Protocolos de transporte: TCP y UDP | 2h |
| [03_http_protocols.md](01_fundamentos_redes/03_http_protocols.md) | HTTP/1.1, HTTP/2, gRPC | 3h |
| [04_tls_mtls.md](01_fundamentos_redes/04_tls_mtls.md) | TLS, mTLS y certificados | 2h |

### Módulo 2: Proxies - Conceptos Fundamentales

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [01_que_es_proxy.md](02_proxies_conceptos/01_que_es_proxy.md) | Qué es un proxy y tipos | 2h |
| [02_proxy_l4.md](02_proxies_conceptos/02_proxy_l4.md) | Proxy Layer 4 en profundidad | 2h |
| [03_proxy_l7.md](02_proxies_conceptos/03_proxy_l7.md) | Proxy Layer 7 en profundidad | 2h |
| [04_comparativa.md](02_proxies_conceptos/04_comparativa.md) | L4 vs L7: cuándo usar cada uno | 1h |

### Módulo 3: Arquitectura de Envoy

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [01_vision_general.md](03_envoy_arquitectura/01_vision_general.md) | Visión general de Envoy | 2h |
| [02_threading_model.md](03_envoy_arquitectura/02_threading_model.md) | Modelo de threading y event loop | 3h |
| [03_life_of_request.md](03_envoy_arquitectura/03_life_of_request.md) | Vida de un request (completo) | 4h |
| [04_filter_chains.md](03_envoy_arquitectura/04_filter_chains.md) | Network y HTTP filter chains | 3h |
| [05_cluster_management.md](03_envoy_arquitectura/05_cluster_management.md) | Clusters, load balancing, health checks | 3h |
| [06_xds_protocol.md](03_envoy_arquitectura/06_xds_protocol.md) | xDS protocol y configuración dinámica | 3h |

#### Code Analysis (Deep Dive)

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [conn_manager_deep_dive.md](03_envoy_arquitectura/code_analysis/conn_manager_deep_dive.md) | HTTP Connection Manager internals | 4h |
| [router_filter_analysis.md](03_envoy_arquitectura/code_analysis/router_filter_analysis.md) | Router filter implementation | 3h |
| [load_balancer_internals.md](03_envoy_arquitectura/code_analysis/load_balancer_internals.md) | Load balancer algorithms | 3h |

### Módulo 4: Arquitectura de ztunnel

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [01_ambient_mode_context.md](04_ztunnel_arquitectura/01_ambient_mode_context.md) | Istio ambient mode y contexto | 2h |
| [02_threading_tokio.md](04_ztunnel_arquitectura/02_threading_tokio.md) | Threading model con Tokio | 3h |
| [03_hbone_protocol.md](04_ztunnel_arquitectura/03_hbone_protocol.md) | HBONE protocol deep dive | 3h |
| [04_traffic_redirection.md](04_ztunnel_arquitectura/04_traffic_redirection.md) | Redirección de tráfico con iptables | 2h |

#### Code Analysis (Deep Dive)

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [proxy_module.md](04_ztunnel_arquitectura/code_analysis/proxy_module.md) | Módulo proxy/ análisis | 4h |
| [xds_integration.md](04_ztunnel_arquitectura/code_analysis/xds_integration.md) | Integración xDS | 2h |
| [identity_management.md](04_ztunnel_arquitectura/code_analysis/identity_management.md) | Gestión de identidades SPIFFE | 2h |

### Módulo 5: Comparativa

| Documento | Descripción | Tiempo |
|-----------|-------------|--------|
| [01_envoy_vs_ztunnel.md](05_comparativa/01_envoy_vs_ztunnel.md) | Comparación detallada | 2h |
| [02_ambient_mode_completo.md](05_comparativa/02_ambient_mode_completo.md) | Arquitectura ambient completa | 2h |
| [03_decision_framework.md](05_comparativa/03_decision_framework.md) | Marco de decisión | 1h |

### Módulo 6: Avanzado

#### Envoy
| Documento | Descripción |
|-----------|-------------|
| [custom_filters.md](06_avanzado/envoy/custom_filters.md) | Escribir filtros personalizados |
| [hot_restart.md](06_avanzado/envoy/hot_restart.md) | Mecanismo de hot restart |
| [performance_tuning.md](06_avanzado/envoy/performance_tuning.md) | Tuning de rendimiento |

#### ztunnel
| Documento | Descripción |
|-----------|-------------|
| [rust_async_patterns.md](06_avanzado/ztunnel/rust_async_patterns.md) | Patrones async en Rust |
| [certificate_rotation.md](06_avanzado/ztunnel/certificate_rotation.md) | Rotación de certificados |

### Ejercicios Prácticos

#### Envoy
| Ejercicio | Descripción |
|-----------|-------------|
| [01_compilar_ejecutar.md](ejercicios/envoy/01_compilar_ejecutar.md) | Compilar y ejecutar Envoy |
| [02_trazar_request.md](ejercicios/envoy/02_trazar_request.md) | Trazar un request con logging |
| [03_modificar_filtro.md](ejercicios/envoy/03_modificar_filtro.md) | Modificar un filtro existente |

#### ztunnel
| Ejercicio | Descripción |
|-----------|-------------|
| [01_build_run.md](ejercicios/ztunnel/01_build_run.md) | Compilar y ejecutar ztunnel |
| [02_analizar_conexion.md](ejercicios/ztunnel/02_analizar_conexion.md) | Analizar flujo de conexión |

---

## Plan de Estudio Recomendado

### Semana 1: Fundamentos
- Completar Módulos 1 y 2
- Ejercicios conceptuales

### Semanas 2-3: Envoy
- Completar Módulo 3
- Ejercicios prácticos de Envoy
- Code analysis según interés

### Semanas 3-4: ztunnel
- Completar Módulo 4
- Ejercicios prácticos de ztunnel
- Familiarizarse con Rust async

### Semana 5: Integración
- Completar Módulo 5
- Experimentar con ambient mode

### Ongoing: Especialización
- Módulo 6 según necesidades
- Contribuciones al código

---

## Recursos Adicionales

- [recursos/referencias.md](recursos/referencias.md) - Links y documentación oficial
- [recursos/diagramas/](recursos/diagramas/) - Diagramas de arquitectura
- [recursos/config_examples/](recursos/config_examples/) - Ejemplos de configuración

---

## Convenciones

### Idioma
- **Conceptos**: Español
- **Términos técnicos**: Inglés (con definición la primera vez)
- **Código**: Inglés (como en el código fuente)

### Referencias a Código
Formato: `archivo.cc:línea` o `archivo.cc:línea-línea`

Ejemplo:
```
source/common/http/conn_manager_impl.cc:394-420
```

### Diagramas
Se usa sintaxis Mermaid y ASCII donde corresponda.

---

## Cómo Usar Este Material

1. **Lectura secuencial**: Los módulos están ordenados para construir conocimiento progresivamente
2. **Código a mano**: Abre los archivos referenciados mientras lees
3. **Ejercicios**: No los saltes - la práctica consolida el conocimiento
4. **Preguntas**: Anota dudas para investigar o preguntar
5. **Iteración**: Vuelve a módulos anteriores cuando sea necesario

---

*Última actualización: 2025-12-04*
