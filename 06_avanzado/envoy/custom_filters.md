# Escribiendo Custom HTTP Filters en Envoy

---
**Módulo**: 6 - Avanzado (Envoy)
**Tema**: Desarrollo de HTTP Filters personalizados
**Tiempo estimado**: 4 horas
**Prerrequisitos**: Módulo 3 completo
---

## Objetivos de Aprendizaje

Al completar este documento:
- Entenderás la arquitectura de filters de Envoy
- Conocerás las interfaces que debe implementar un filter
- Podrás crear un HTTP filter básico
- Sabrás cómo registrar y configurar tu filter

---

## 1. Arquitectura de Filters

### 1.1 Tipos de Filters

```
┌─────────────────────────────────────────────────────────────────┐
│                    Filter Types in Envoy                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LISTENER FILTERS                                              │
│  ─────────────────                                              │
│  • Se ejecutan durante connection accept                       │
│  • Ejemplo: TLS Inspector, HTTP Inspector                      │
│  • Path: source/extensions/filters/listener/                   │
│                                                                 │
│  NETWORK FILTERS (L4)                                          │
│  ────────────────────                                           │
│  • Se ejecutan sobre raw TCP/UDP                               │
│  • Ejemplo: TCP Proxy, Redis Proxy                             │
│  • Path: source/extensions/filters/network/                    │
│                                                                 │
│  HTTP FILTERS (L7)                                             │
│  ─────────────────                                              │
│  • Se ejecutan sobre HTTP requests/responses                   │
│  • Ejemplo: Router, CORS, JWT Auth                             │
│  • Path: source/extensions/filters/http/                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 HTTP Filter Chain

```
┌─────────────────────────────────────────────────────────────────┐
│                   HTTP Filter Chain                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request Path (decode):                                        │
│                                                                 │
│  Downstream ──> [Filter 1] ──> [Filter 2] ──> [Filter N] ──>   │
│                 decodeHeaders  decodeHeaders  decodeHeaders     │
│                 decodeData     decodeData     decodeData        │
│                 decodeTrailers decodeTrailers decodeTrailers    │
│                                                                 │
│                                                       │         │
│                                                       ▼         │
│                                                   Upstream      │
│                                                       │         │
│                                                       │         │
│  Response Path (encode):                              ▼         │
│                                                                 │
│  Downstream <── [Filter 1] <── [Filter 2] <── [Filter N] <──   │
│                 encodeHeaders  encodeHeaders  encodeHeaders     │
│                 encodeData     encodeData     encodeData        │
│                 encodeTrailers encodeTrailers encodeTrailers    │
│                                                                 │
│  Filter puede:                                                 │
│  • Continuar (Continue)                                        │
│  • Pausar (StopIteration)                                      │
│  • Responder directamente (sendLocalReply)                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Interfaces Principales

### 2.1 StreamDecoderFilter (Request Path)

```cpp
// source/common/http/filter_manager.h

class StreamDecoderFilter {
public:
    virtual ~StreamDecoderFilter() = default;

    // Llamado cuando llegan request headers
    virtual FilterHeadersStatus decodeHeaders(
        RequestHeaderMap& headers,
        bool end_stream) PURE;

    // Llamado cuando llega request body
    virtual FilterDataStatus decodeData(
        Buffer::Instance& data,
        bool end_stream) PURE;

    // Llamado cuando llegan request trailers
    virtual FilterTrailersStatus decodeTrailers(
        RequestTrailerMap& trailers) PURE;
};

// FilterHeadersStatus puede ser:
// - Continue: continuar al siguiente filter
// - StopIteration: pausar, esperar async operation
// - StopAllIterationAndBuffer: pausar y bufferar
// - StopAllIterationAndWatermark: pausar con watermarks
```

### 2.2 StreamEncoderFilter (Response Path)

```cpp
class StreamEncoderFilter {
public:
    virtual ~StreamEncoderFilter() = default;

    // Llamado cuando llegan response headers
    virtual FilterHeadersStatus encodeHeaders(
        ResponseHeaderMap& headers,
        bool end_stream) PURE;

    // Llamado cuando llega response body
    virtual FilterDataStatus encodeData(
        Buffer::Instance& data,
        bool end_stream) PURE;

    // Llamado cuando llegan response trailers
    virtual FilterTrailersStatus encodeTrailers(
        ResponseTrailerMap& trailers) PURE;
};
```

### 2.3 StreamFilter (Ambos)

```cpp
// Combina decoder y encoder
class StreamFilter : public StreamDecoderFilter,
                     public StreamEncoderFilter {
    // Implementa ambas interfaces
};
```

---

## 3. Creando un Filter Simple

### 3.1 Estructura del Filter

Vamos a crear un filter que añade un header personalizado:

```
source/extensions/filters/http/my_header/
├── BUILD
├── config.cc           # Factory de configuración
├── config.h
├── my_header_filter.cc # Implementación del filter
├── my_header_filter.h
└── my_header.proto     # Definición de configuración
```

### 3.2 Definición de Configuración (Proto)

```protobuf
// api/envoy/extensions/filters/http/my_header/v3/my_header.proto

syntax = "proto3";

package envoy.extensions.filters.http.my_header.v3;

option java_package = "io.envoyproxy.envoy.extensions.filters.http.my_header.v3";

message MyHeader {
    // Nombre del header a añadir
    string header_name = 1;

    // Valor del header
    string header_value = 2;

    // Añadir en request (true) o response (false)
    bool add_to_request = 3;
}
```

### 3.3 Implementación del Filter

```cpp
// source/extensions/filters/http/my_header/my_header_filter.h

#pragma once

#include "envoy/http/filter.h"
#include "source/common/common/logger.h"

namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace MyHeader {

// Configuración parseada
struct MyHeaderConfig {
    std::string header_name;
    std::string header_value;
    bool add_to_request;
};

using MyHeaderConfigSharedPtr = std::shared_ptr<MyHeaderConfig>;

class MyHeaderFilter : public Http::StreamFilter,
                       Logger::Loggable<Logger::Id::filter> {
public:
    explicit MyHeaderFilter(MyHeaderConfigSharedPtr config);

    // StreamDecoderFilter
    Http::FilterHeadersStatus decodeHeaders(
        Http::RequestHeaderMap& headers,
        bool end_stream) override;

    Http::FilterDataStatus decodeData(
        Buffer::Instance& data,
        bool end_stream) override {
        return Http::FilterDataStatus::Continue;
    }

    Http::FilterTrailersStatus decodeTrailers(
        Http::RequestTrailerMap& trailers) override {
        return Http::FilterTrailersStatus::Continue;
    }

    void setDecoderFilterCallbacks(
        Http::StreamDecoderFilterCallbacks& callbacks) override {
        decoder_callbacks_ = &callbacks;
    }

    // StreamEncoderFilter
    Http::FilterHeadersStatus encodeHeaders(
        Http::ResponseHeaderMap& headers,
        bool end_stream) override;

    Http::FilterDataStatus encodeData(
        Buffer::Instance& data,
        bool end_stream) override {
        return Http::FilterDataStatus::Continue;
    }

    Http::FilterTrailersStatus encodeTrailers(
        Http::ResponseTrailerMap& trailers) override {
        return Http::FilterTrailersStatus::Continue;
    }

    void setEncoderFilterCallbacks(
        Http::StreamEncoderFilterCallbacks& callbacks) override {
        encoder_callbacks_ = &callbacks;
    }

    // Filter chain completion
    void onDestroy() override {}

private:
    MyHeaderConfigSharedPtr config_;
    Http::StreamDecoderFilterCallbacks* decoder_callbacks_{};
    Http::StreamEncoderFilterCallbacks* encoder_callbacks_{};
};

} // namespace MyHeader
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

```cpp
// source/extensions/filters/http/my_header/my_header_filter.cc

#include "source/extensions/filters/http/my_header/my_header_filter.h"

namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace MyHeader {

MyHeaderFilter::MyHeaderFilter(MyHeaderConfigSharedPtr config)
    : config_(std::move(config)) {}

Http::FilterHeadersStatus MyHeaderFilter::decodeHeaders(
    Http::RequestHeaderMap& headers,
    bool /* end_stream */) {

    if (config_->add_to_request) {
        ENVOY_LOG(debug, "Adding header {} = {} to request",
                  config_->header_name, config_->header_value);

        headers.addCopy(
            Http::LowerCaseString(config_->header_name),
            config_->header_value);
    }

    return Http::FilterHeadersStatus::Continue;
}

Http::FilterHeadersStatus MyHeaderFilter::encodeHeaders(
    Http::ResponseHeaderMap& headers,
    bool /* end_stream */) {

    if (!config_->add_to_request) {
        ENVOY_LOG(debug, "Adding header {} = {} to response",
                  config_->header_name, config_->header_value);

        headers.addCopy(
            Http::LowerCaseString(config_->header_name),
            config_->header_value);
    }

    return Http::FilterHeadersStatus::Continue;
}

} // namespace MyHeader
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

### 3.4 Factory de Configuración

```cpp
// source/extensions/filters/http/my_header/config.h

#pragma once

#include "envoy/extensions/filters/http/my_header/v3/my_header.pb.h"
#include "source/extensions/filters/http/common/factory_base.h"

namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace MyHeader {

class MyHeaderFilterFactory
    : public Common::FactoryBase<
          envoy::extensions::filters::http::my_header::v3::MyHeader> {
public:
    MyHeaderFilterFactory() : FactoryBase("envoy.filters.http.my_header") {}

private:
    Http::FilterFactoryCb createFilterFactoryFromProtoTyped(
        const envoy::extensions::filters::http::my_header::v3::MyHeader& config,
        const std::string& stats_prefix,
        Server::Configuration::FactoryContext& context) override;
};

} // namespace MyHeader
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

```cpp
// source/extensions/filters/http/my_header/config.cc

#include "source/extensions/filters/http/my_header/config.h"
#include "source/extensions/filters/http/my_header/my_header_filter.h"

namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace MyHeader {

Http::FilterFactoryCb MyHeaderFilterFactory::createFilterFactoryFromProtoTyped(
    const envoy::extensions::filters::http::my_header::v3::MyHeader& proto_config,
    const std::string& /* stats_prefix */,
    Server::Configuration::FactoryContext& /* context */) {

    auto config = std::make_shared<MyHeaderConfig>();
    config->header_name = proto_config.header_name();
    config->header_value = proto_config.header_value();
    config->add_to_request = proto_config.add_to_request();

    return [config](Http::FilterChainFactoryCallbacks& callbacks) -> void {
        callbacks.addStreamFilter(std::make_shared<MyHeaderFilter>(config));
    };
}

// Registrar factory
REGISTER_FACTORY(MyHeaderFilterFactory,
                 Server::Configuration::NamedHttpFilterConfigFactory);

} // namespace MyHeader
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

### 3.5 BUILD file

```python
# source/extensions/filters/http/my_header/BUILD

load(
    "//bazel:envoy_build_system.bzl",
    "envoy_cc_extension",
    "envoy_cc_library",
    "envoy_extension_package",
)

envoy_extension_package()

envoy_cc_library(
    name = "my_header_filter_lib",
    srcs = ["my_header_filter.cc"],
    hdrs = ["my_header_filter.h"],
    deps = [
        "//envoy/http:filter_interface",
        "//source/common/common:logger_lib",
        "//source/common/http:header_map_lib",
    ],
)

envoy_cc_extension(
    name = "config",
    srcs = ["config.cc"],
    hdrs = ["config.h"],
    deps = [
        ":my_header_filter_lib",
        "//source/extensions/filters/http/common:factory_base_lib",
        "@envoy_api//envoy/extensions/filters/http/my_header/v3:pkg_cc_proto",
    ],
)
```

---

## 4. Registrar la Extensión

### 4.1 En extensions_build_config.bzl

```python
# source/extensions/extensions_build_config.bzl

# Añadir en la sección de HTTP filters
EXTENSIONS = {
    # ... otras extensiones ...

    "envoy.filters.http.my_header":
        "//source/extensions/filters/http/my_header:config",

    # ... otras extensiones ...
}
```

### 4.2 Configuración de Uso

```yaml
# envoy.yaml

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
                route:
                  cluster: backend
          http_filters:
          # Nuestro filter custom!
          - name: envoy.filters.http.my_header
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.my_header.v3.MyHeader
              header_name: "X-Custom-Header"
              header_value: "added-by-my-filter"
              add_to_request: true
          # Router siempre al final
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

---

## 5. Patrones Avanzados

### 5.1 Filter con Estado Async

```cpp
// Filter que hace llamada externa

Http::FilterHeadersStatus MyAsyncFilter::decodeHeaders(
    Http::RequestHeaderMap& headers,
    bool end_stream) {

    // Iniciar llamada async
    auto callback = [this](bool success) {
        if (success) {
            // Continuar procesamiento
            decoder_callbacks_->continueDecoding();
        } else {
            // Responder con error
            decoder_callbacks_->sendLocalReply(
                Http::Code::Forbidden,
                "Async check failed",
                nullptr, absl::nullopt, "async_check_failed");
        }
    };

    external_service_->checkAsync(headers, callback);

    // Pausar mientras esperamos
    return Http::FilterHeadersStatus::StopIteration;
}
```

### 5.2 Filter con Buffering

```cpp
// Filter que bufferea todo el body

class BufferingFilter : public Http::StreamFilter {
    Http::FilterDataStatus decodeData(
        Buffer::Instance& data,
        bool end_stream) override {

        // Añadir datos al buffer
        buffer_.add(data);
        data.drain(data.length());

        if (end_stream) {
            // Tenemos todo el body, procesarlo
            processFullBody(buffer_);
            // Pasar el body modificado
            decoder_callbacks_->addDecodedData(buffer_, true);
            return Http::FilterDataStatus::Continue;
        }

        // Seguir buffeando
        return Http::FilterDataStatus::StopIterationAndBuffer;
    }

private:
    Buffer::OwnedImpl buffer_;
};
```

### 5.3 Filter con Métricas

```cpp
class MetricsFilter : public Http::StreamFilter {
public:
    MetricsFilter(Stats::Scope& scope)
        : requests_total_(scope.counterFromString("my_filter.requests_total")),
          requests_blocked_(scope.counterFromString("my_filter.requests_blocked")) {}

    Http::FilterHeadersStatus decodeHeaders(
        Http::RequestHeaderMap& headers,
        bool end_stream) override {

        requests_total_.inc();

        if (shouldBlock(headers)) {
            requests_blocked_.inc();
            decoder_callbacks_->sendLocalReply(
                Http::Code::Forbidden, "Blocked", nullptr, absl::nullopt, "");
            return Http::FilterHeadersStatus::StopIteration;
        }

        return Http::FilterHeadersStatus::Continue;
    }

private:
    Stats::Counter& requests_total_;
    Stats::Counter& requests_blocked_;
};
```

---

## 6. Testing

### 6.1 Unit Test

```cpp
// test/extensions/filters/http/my_header/my_header_filter_test.cc

#include "source/extensions/filters/http/my_header/my_header_filter.h"
#include "test/mocks/http/mocks.h"
#include "gtest/gtest.h"

namespace Envoy {
namespace Extensions {
namespace HttpFilters {
namespace MyHeader {
namespace {

class MyHeaderFilterTest : public testing::Test {
protected:
    void SetUp() override {
        config_ = std::make_shared<MyHeaderConfig>();
        config_->header_name = "x-test-header";
        config_->header_value = "test-value";
        config_->add_to_request = true;

        filter_ = std::make_unique<MyHeaderFilter>(config_);
        filter_->setDecoderFilterCallbacks(decoder_callbacks_);
    }

    MyHeaderConfigSharedPtr config_;
    std::unique_ptr<MyHeaderFilter> filter_;
    NiceMock<Http::MockStreamDecoderFilterCallbacks> decoder_callbacks_;
};

TEST_F(MyHeaderFilterTest, AddsHeaderToRequest) {
    Http::TestRequestHeaderMapImpl headers{
        {":method", "GET"},
        {":path", "/"},
    };

    EXPECT_EQ(Http::FilterHeadersStatus::Continue,
              filter_->decodeHeaders(headers, true));

    EXPECT_EQ("test-value",
              headers.get(Http::LowerCaseString("x-test-header"))[0]->value().getStringView());
}

} // namespace
} // namespace MyHeader
} // namespace HttpFilters
} // namespace Extensions
} // namespace Envoy
```

### 6.2 Integration Test

```cpp
// test/extensions/filters/http/my_header/my_header_integration_test.cc

class MyHeaderIntegrationTest : public HttpIntegrationTest {
public:
    MyHeaderIntegrationTest()
        : HttpIntegrationTest(Http::CodecType::HTTP1, Network::Address::IpVersion::v4) {}

    void SetUp() override {
        config_helper_.addConfigModifier(
            [](envoy::config::bootstrap::v3::Bootstrap& bootstrap) {
                // Añadir nuestro filter a la config
            });
        HttpIntegrationTest::initialize();
    }
};

TEST_F(MyHeaderIntegrationTest, HeaderIsAdded) {
    codec_client_ = makeHttpConnection(lookupPort("http"));

    auto response = sendRequestAndWaitForResponse(
        default_request_headers_, 0,
        default_response_headers_, 0);

    // Verificar que el upstream recibió el header
    EXPECT_EQ("test-value",
              upstream_request_->headers()
                  .get(Http::LowerCaseString("x-test-header"))[0]
                  ->value()
                  .getStringView());
}
```

---

## 7. Debugging Tips

### 7.1 Logging

```cpp
// Usar ENVOY_LOG para debugging
ENVOY_LOG(debug, "Processing request: method={} path={}",
          headers.getMethodValue(),
          headers.getPathValue());

ENVOY_LOG(trace, "Full headers dump:");
headers.iterate([](const Http::HeaderEntry& header) -> Http::HeaderMap::Iterate {
    ENVOY_LOG(trace, "  {}: {}", header.key().getStringView(),
              header.value().getStringView());
    return Http::HeaderMap::Iterate::Continue;
});
```

### 7.2 Stats

```cpp
// Añadir stats para observabilidad
scope.counterFromString("my_filter.requests_total").inc();
scope.gaugeFromString("my_filter.active_requests",
                      Stats::Gauge::ImportMode::Accumulate).inc();
scope.histogramFromString("my_filter.latency_ms",
                          Stats::Histogram::Unit::Milliseconds)
    .recordValue(latency_ms);
```

---

## 8. Autoevaluación

1. ¿Cuáles son los 3 tipos de filters en Envoy?
2. ¿Qué interfaces debe implementar un HTTP filter?
3. ¿Qué significa retornar `StopIteration` en `decodeHeaders`?
4. ¿Cómo registras un filter custom en Envoy?
5. ¿Cómo harías un filter que hace una llamada async a un servicio externo?

---

## 9. Referencias

| Recurso | Descripción |
|---------|-------------|
| `source/extensions/filters/http/buffer/` | Ejemplo de filter simple |
| `source/extensions/filters/http/cors/` | Filter con config más compleja |
| `source/extensions/filters/http/ext_authz/` | Filter con llamadas externas |
| [Envoy Filter Development](https://www.envoyproxy.io/docs/envoy/latest/extending/extending.html) | Documentación oficial |

---

**Siguiente**: [hot_restart.md](hot_restart.md) - Hot Restart Mechanism

