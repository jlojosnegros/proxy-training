# Ejercicio: Compilar y Ejecutar Envoy

---

**Tipo**: Ejercicio práctico
**Tiempo estimado**: 2-3 horas
**Prerrequisitos**: Módulo 3 completo

---

## Objetivo

Compilar Envoy desde el código fuente y ejecutarlo con una configuración básica.

---

## 1. Preparación del Entorno

### 1.1 Verificar Requisitos

```bash
# Verificar Bazel/Bazelisk
bazel --version
# Debería mostrar: bazel X.X.X

# Si no está instalado:
# Fedora:
sudo wget -O /usr/local/bin/bazel \
  https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-amd64
sudo chmod +x /usr/local/bin/bazel

# Verificar dependencias (Fedora)
rpm -q aspell-en libatomic libstdc++ libstdc++-static libtool lld patch python3-pip

# Instalar si faltan
sudo dnf install -y aspell-en libatomic libstdc++ libstdc++-static libtool lld patch python3-pip
```

### 1.2 Clonar Repositorio (si no lo tienes)

```bash
# Ya deberías tenerlo en:
cd /home/jojosneg/source/redhat/envoy/upstream/main

# Verificar
git status
git log --oneline -5
```

---

## 2. Configurar Compilación

### 2.1 Setup de Clang (Recomendado)

```bash
# Si tienes Clang instalado:
./bazel/setup_clang.sh /usr/lib64/llvm

# Añadir a configuración
echo "build --config=clang" >> user.bazelrc
```

### 2.2 Verificar Configuración

```bash
# Ver opciones de build disponibles
cat .bazelrc

# Ver tu configuración local
cat user.bazelrc
```

---

## 3. Compilar Envoy

### 3.1 Build Rápido (fastbuild)

```bash
# Build básico - más rápido pero sin optimizaciones
bazel build //source/exe:envoy-static

# Tiempo estimado: 30-60 minutos primera vez
# Builds posteriores son más rápidos (caché)
```

### 3.2 Build Optimizado (opcional)

```bash
# Build optimizado para producción
bazel build -c opt //source/exe:envoy-static

# Más lento pero mejor performance
```

### 3.3 Verificar Build

```bash
# Verificar que se creó el binario
ls -la bazel-bin/source/exe/envoy-static

# Ver tamaño
du -h bazel-bin/source/exe/envoy-static
# Típicamente 100-200MB
```

---

## 4. Crear Configuración de Prueba

### 4.1 Crear Archivo de Configuración

Crea un archivo `test_config.yaml`:

```yaml
# test_config.yaml

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
                codec_type: AUTO
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          direct_response:
                            status: 200
                            body:
                              inline_string: "Hello from Envoy!\n"
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

### 4.2 Validar Configuración

```bash
# Validar que la config es correcta
./bazel-bin/source/exe/envoy-static --mode validate -c test_config.yaml

# Debería mostrar: configuration is valid
```

---

## 5. Ejecutar Envoy

### 5.1 Iniciar Envoy

```bash
# Ejecutar en foreground
./bazel-bin/source/exe/envoy-static -c test_config.yaml

# Logs deberían mostrar:
# [info] starting main dispatch loop
# [info] [listener] listener listening on 0.0.0.0:8080
```

### 5.2 Probar Funcionamiento

En otra terminal:

```bash
# Probar el listener
curl http://localhost:8080/
# Debería responder: Hello from Envoy!

# Probar admin interface
curl http://localhost:15000/server_info | jq .

# Ver estadísticas
curl http://localhost:15000/stats | head -20

# Ver listeners
curl http://localhost:15000/listeners

# Ver clusters
curl http://localhost:15000/clusters
```

---

## 6. Explorar Admin Interface

### 6.1 Endpoints Útiles

| Endpoint            | Descripción                 |
| ------------------- | --------------------------- |
| `/server_info`      | Información del servidor    |
| `/stats`            | Todas las estadísticas      |
| `/stats/prometheus` | Stats en formato Prometheus |
| `/clusters`         | Estado de clusters          |
| `/listeners`        | Listeners configurados      |
| `/config_dump`      | Configuración completa      |
| `/logging`          | Ver/cambiar niveles de log  |
| `/ready`            | Readiness check             |

### 6.2 Cambiar Log Level

```bash
# Ver nivel actual
curl http://localhost:15000/logging

# Cambiar a debug
curl -X POST "http://localhost:15000/logging?level=debug"

# Cambiar solo para un componente
curl -X POST "http://localhost:15000/logging?upstream=debug"
```

---

## 7. Ejercicio: Añadir Upstream Real

### 7.1 Modificar Configuración

Actualiza `test_config.yaml` para añadir un upstream:

```yaml
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
                codec_type: AUTO
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: httpbin
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: httpbin
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: httpbin
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: httpbin.org
                      port_value: 80
```

### 7.2 Probar

```bash
# Restart Envoy con nueva config
./bazel-bin/source/exe/envoy-static -c test_config.yaml

# Probar proxy a httpbin
curl http://localhost:8080/get

# Debería mostrar response de httpbin.org
```

---

## 8. Checkpoints de Verificación

- [ ] Bazel instalado y funcionando
- [ ] Dependencias del sistema instaladas
- [ ] Build de Envoy completado exitosamente
- [ ] Configuración básica validada
- [ ] Envoy ejecutándose y respondiendo en :8080
- [ ] Admin interface accesible en :15000
- [ ] Proxy a upstream externo funcionando

---

## 9. Troubleshooting

### Build Falla

```bash
# Limpiar caché y reintentar
bazel clean --expunge
bazel build //source/exe:envoy-static

# Ver errores detallados
bazel build --sandbox_debug //source/exe:envoy-static
```

### Envoy No Inicia

```bash
# Verificar permisos de puerto
sudo ss -tlnp | grep 8080

# Ejecutar con más logs
./bazel-bin/source/exe/envoy-static -c test_config.yaml -l debug
```

### Config Inválida

```bash
# Validar config con más detalle
./bazel-bin/source/exe/envoy-static --mode validate -c test_config.yaml --log-level debug
```

---

## 10. Siguientes Pasos

Una vez completado este ejercicio, continúa con:

- [02_trazar_request.md](02_trazar_request.md) - Trazar un Request

---

**Tiempo total estimado**: 2-3 horas (incluyendo tiempo de build)
