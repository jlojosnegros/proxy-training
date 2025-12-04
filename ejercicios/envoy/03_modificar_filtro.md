# Ejercicio: Modificar un Filtro HTTP

---
**Tipo**: Ejercicio práctico
**Tiempo estimado**: 2-3 horas
**Prerrequisitos**: Ejercicios 01 y 02 completados, Módulo 6 (custom filters)
---

## Objetivo

Modificar un filtro HTTP existente en Envoy para entender el flujo de desarrollo.

---

## 1. Explorar un Filtro Simple

### 1.1 Elegir el Filtro Buffer

El filtro buffer es simple y bueno para aprender:

```bash
cd /home/jojosneg/source/redhat/envoy/upstream/main

# Ver estructura del filtro
ls -la source/extensions/filters/http/buffer/

# Archivos principales:
# - buffer_filter.h     # Declaración
# - buffer_filter.cc    # Implementación
# - config.h/cc         # Factory de configuración
```

### 1.2 Leer el Código

```bash
# Ver la cabecera
cat source/extensions/filters/http/buffer/buffer_filter.h

# Ver la implementación
cat source/extensions/filters/http/buffer/buffer_filter.cc
```

**Puntos clave a observar:**
- Herencia de `StreamDecoderFilter`
- Métodos `decodeHeaders`, `decodeData`
- Uso de `FilterHeadersStatus` y `FilterDataStatus`

---

## 2. Añadir Logging al Filtro

### 2.1 Modificar buffer_filter.cc

Abre `source/extensions/filters/http/buffer/buffer_filter.cc` y añade logging:

```cpp
// Después de los includes existentes
#include "source/common/common/logger.h"

// En el método decodeHeaders, añadir al inicio:
Http::FilterHeadersStatus BufferFilter::decodeHeaders(Http::RequestHeaderMap& headers,
                                                       bool end_stream) {
    // AÑADIR ESTA LÍNEA:
    ENVOY_LOG(info, "BufferFilter: processing headers, path={}, method={}",
              headers.getPathValue(), headers.getMethodValue());

    // ... resto del código existente ...
}

// En decodeData, añadir al inicio:
Http::FilterDataStatus BufferFilter::decodeData(Buffer::Instance& data, bool end_stream) {
    // AÑADIR ESTA LÍNEA:
    ENVOY_LOG(info, "BufferFilter: received {} bytes, end_stream={}",
              data.length(), end_stream);

    // ... resto del código existente ...
}
```

### 2.2 Compilar

```bash
# Compilar solo el filtro (más rápido)
bazel build //source/extensions/filters/http/buffer:config

# O compilar Envoy completo
bazel build //source/exe:envoy-static
```

### 2.3 Probar

Crea una configuración que use el filtro:

```yaml
# buffer_test.yaml
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
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                direct_response:
                  status: 200
                  body:
                    inline_string: "OK"
          http_filters:
          # Nuestro filtro modificado
          - name: envoy.filters.http.buffer
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.buffer.v3.Buffer
              max_request_bytes: 1048576
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

```bash
# Ejecutar
./bazel-bin/source/exe/envoy-static -c buffer_test.yaml -l info

# Hacer request
curl http://localhost:8080/test

# Observar logs con nuestro mensaje
```

---

## 3. Añadir un Header Custom

### 3.1 Modificar para Añadir Header

En `buffer_filter.cc`, modifica `decodeHeaders`:

```cpp
Http::FilterHeadersStatus BufferFilter::decodeHeaders(Http::RequestHeaderMap& headers,
                                                       bool end_stream) {
    ENVOY_LOG(info, "BufferFilter: processing headers");

    // AÑADIR: Agregar header personalizado
    headers.addCopy(Http::LowerCaseString("x-buffer-filter"), "processed");

    // ... resto del código ...
}
```

### 3.2 Verificar

```bash
# Recompilar
bazel build //source/exe:envoy-static

# Ejecutar
./bazel-bin/source/exe/envoy-static -c buffer_test.yaml

# Verificar header (usando httpbin para ver headers)
# Cambiar config para usar httpbin como upstream y verificar
```

---

## 4. Añadir una Estadística

### 4.1 Definir Stats

Modifica `buffer_filter.h` para añadir stats:

```cpp
// En la sección de includes
#include "envoy/stats/scope.h"
#include "envoy/stats/stats_macros.h"

// Definir stats struct
#define ALL_BUFFER_FILTER_STATS(COUNTER)                                                           \
  COUNTER(requests_buffered)

struct BufferFilterStats {
  ALL_BUFFER_FILTER_STATS(GENERATE_COUNTER_STRUCT)
};

// En la clase BufferFilter, añadir member:
private:
  BufferFilterStats stats_;
```

### 4.2 Inicializar Stats

En el constructor o config, inicializar:

```cpp
BufferFilterStats generateStats(const std::string& prefix, Stats::Scope& scope) {
  return {ALL_BUFFER_FILTER_STATS(POOL_COUNTER_PREFIX(scope, prefix))};
}
```

### 4.3 Usar Stats

```cpp
Http::FilterHeadersStatus BufferFilter::decodeHeaders(...) {
    // Incrementar contador
    stats_.requests_buffered_.inc();
    // ...
}
```

### 4.4 Verificar

```bash
# Recompilar y ejecutar
bazel build //source/exe:envoy-static
./bazel-bin/source/exe/envoy-static -c buffer_test.yaml

# Hacer requests
curl http://localhost:8080/test

# Ver stats
curl http://localhost:15000/stats | grep buffer
# Debería mostrar: http.ingress_http.buffer.requests_buffered: 1
```

---

## 5. Ejecutar Tests del Filtro

### 5.1 Encontrar Tests

```bash
ls test/extensions/filters/http/buffer/

# Archivo principal: buffer_filter_test.cc
```

### 5.2 Ejecutar Tests

```bash
# Ejecutar tests del filtro
bazel test //test/extensions/filters/http/buffer:buffer_filter_test

# Con output detallado
bazel test //test/extensions/filters/http/buffer:buffer_filter_test --test_output=all
```

### 5.3 Añadir Test para Nuestra Modificación

Abre `test/extensions/filters/http/buffer/buffer_filter_test.cc` y añade:

```cpp
TEST_F(BufferFilterTest, AddsCustomHeader) {
  // Setup
  EXPECT_CALL(decoder_callbacks_, decodingBuffer()).Times(0);

  Http::TestRequestHeaderMapImpl request_headers{
      {":path", "/test"},
      {":method", "GET"},
  };

  // Act
  EXPECT_EQ(Http::FilterHeadersStatus::Continue,
            filter_->decodeHeaders(request_headers, true));

  // Assert - verificar que se añadió el header
  EXPECT_EQ("processed",
            request_headers.get(Http::LowerCaseString("x-buffer-filter"))[0]->value().getStringView());
}
```

### 5.4 Ejecutar Nuevo Test

```bash
bazel test //test/extensions/filters/http/buffer:buffer_filter_test \
  --test_arg=--gtest_filter="*AddsCustomHeader*"
```

---

## 6. Checkpoints de Verificación

- [ ] Código del filtro buffer explorado
- [ ] Logging añadido y funcionando
- [ ] Header custom añadido
- [ ] Stats añadidas y visibles
- [ ] Tests existentes pasando
- [ ] Nuevo test escrito y pasando

---

## 7. Revertir Cambios

```bash
# Revertir cambios al código original
cd /home/jojosneg/source/redhat/envoy/upstream/main
git checkout source/extensions/filters/http/buffer/
git checkout test/extensions/filters/http/buffer/

# Verificar
git status
```

---

## 8. Reflexión

Después de este ejercicio, deberías entender:

1. **Estructura de un filtro HTTP**
   - Header file con declaraciones
   - Implementation file con lógica
   - Config factory para crear instancias

2. **Métodos principales**
   - `decodeHeaders()` - procesar request headers
   - `decodeData()` - procesar request body
   - `encodeHeaders()` - procesar response headers
   - `encodeData()` - procesar response body

3. **Patrones comunes**
   - Uso de `ENVOY_LOG` para logging
   - Stats con macros
   - Testing con mocks

---

## 9. Siguiente Paso

Para profundizar más, considera:
- Crear un filtro completamente nuevo (ver Módulo 6)
- Explorar filtros más complejos como `router` o `ext_authz`
- Contribuir una mejora al proyecto upstream

---

**Siguiente ejercicio**: [../ztunnel/01_build_run.md](../ztunnel/01_build_run.md) - Compilar y Ejecutar ztunnel

