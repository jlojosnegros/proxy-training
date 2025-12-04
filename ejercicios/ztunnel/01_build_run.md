# Ejercicio: Compilar y Ejecutar ztunnel

---
**Tipo**: Ejercicio práctico
**Tiempo estimado**: 1-2 horas
**Prerrequisitos**: Módulo 4 completo, Rust instalado
---

## Objetivo

Compilar ztunnel desde el código fuente y ejecutar tests básicos.

---

## 1. Preparación del Entorno

### 1.1 Verificar Rust

```bash
# Verificar instalación de Rust
rustc --version
# Debería ser 1.70+

cargo --version

# Si no está instalado:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

### 1.2 Ir al Repositorio

```bash
cd /home/jojosneg/source/redhat/ztunnel/upstream/master

# Verificar
git status
ls -la
```

### 1.3 Verificar Dependencias

```bash
# En Fedora, algunas dependencias de sistema:
sudo dnf install -y openssl-devel protobuf-compiler
```

---

## 2. Explorar la Estructura

### 2.1 Archivos Principales

```bash
# Ver estructura
tree -L 2 src/

# Archivos importantes:
# src/main.rs          - Entry point
# src/proxy/           - Core proxy logic
# src/xds/            - xDS client
# src/identity/       - Certificate management
# Cargo.toml          - Dependencies

# Documentación
cat ARCHITECTURE.md
cat README.md
```

### 2.2 Ver Cargo.toml

```bash
cat Cargo.toml | head -50
```

---

## 3. Compilar ztunnel

### 3.1 Build Debug

```bash
# Build debug (más rápido, sin optimizaciones)
cargo build

# El binario estará en:
ls -la target/debug/ztunnel
```

### 3.2 Build Release

```bash
# Build release (optimizado)
cargo build --release

# Binario optimizado:
ls -la target/release/ztunnel
du -h target/release/ztunnel
```

### 3.3 Verificar Compilación

```bash
# Ver ayuda
./target/debug/ztunnel --help

# Ver versión
./target/debug/ztunnel --version
```

---

## 4. Ejecutar Tests

### 4.1 Tests Unitarios

```bash
# Ejecutar todos los tests
cargo test

# Con output
cargo test -- --nocapture

# Tests específicos
cargo test proxy::
cargo test xds::
```

### 4.2 Tests de un Módulo

```bash
# Solo tests del módulo proxy
cargo test --package ztunnel proxy

# Solo tests del módulo identity
cargo test --package ztunnel identity
```

### 4.3 Test Específico

```bash
# Ejecutar un test específico
cargo test test_name -- --nocapture
```

---

## 5. Explorar el Código

### 5.1 Entry Point

```bash
# Ver main.rs
cat src/main.rs

# Puntos clave:
# - Inicialización de Tokio runtime
# - Separación admin/worker runtimes
# - Conexión a xDS
```

### 5.2 Proxy Module

```bash
# Estructura del proxy
ls src/proxy/

# Archivos principales:
# - mod.rs       - Module exports
# - inbound.rs   - Inbound connection handling
# - outbound.rs  - Outbound connection handling
```

### 5.3 Identity Module

```bash
# Manejo de certificados
ls src/identity/

# Ver cómo se obtienen certs SPIFFE
```

---

## 6. Modo Local (Sin Kubernetes)

### 6.1 Opciones de Configuración

```bash
# Ver variables de entorno
./target/debug/ztunnel --help

# Variables importantes:
# XDS_ADDRESS - Dirección de istiod
# CA_ADDRESS - Dirección de CA
# LOCAL_XDS_PATH - Para xDS desde archivo
# RUST_LOG - Nivel de logging
```

### 6.2 Ejecutar con Logging

```bash
# Ejecutar con debug logging
RUST_LOG=debug ./target/debug/ztunnel

# Solo para ciertos módulos
RUST_LOG=ztunnel::proxy=debug ./target/debug/ztunnel
```

### 6.3 Modo Fake

ztunnel puede ejecutarse en modo "fake" para desarrollo:

```bash
# Si existe un modo de testing local
RUST_LOG=info ./target/debug/ztunnel --local-mode
```

---

## 7. Benchmarks

### 7.1 Ejecutar Benchmarks

```bash
# Si hay benchmarks configurados
cargo bench

# Benchmark específico
cargo bench proxy_throughput
```

---

## 8. Documentación

### 8.1 Generar Docs

```bash
# Generar documentación
cargo doc --no-deps --open

# Esto abrirá la documentación en el navegador
```

### 8.2 Ver Documentación de Módulos

La documentación generada mostrará:
- Estructuras públicas
- Funciones y sus signatures
- Traits implementados

---

## 9. Checkpoints de Verificación

- [ ] Rust instalado y funcionando
- [ ] Repositorio clonado
- [ ] Build debug exitoso
- [ ] Build release exitoso
- [ ] Tests pasando
- [ ] Documentación generada
- [ ] Estructura del código explorada

---

## 10. Troubleshooting

### Build Falla por OpenSSL

```bash
# Instalar dev headers
sudo dnf install openssl-devel

# Limpiar y reintentar
cargo clean
cargo build
```

### Build Falla por Protobuf

```bash
# Instalar protobuf compiler
sudo dnf install protobuf-compiler

cargo build
```

### Tests Fallan

```bash
# Ver tests con más detalle
cargo test -- --nocapture 2>&1 | tee test_output.log

# Buscar errores
grep -i "error\|failed" test_output.log
```

---

## 11. Exploración Adicional

### Ver Cómo se Maneja una Conexión

```bash
# Archivo clave para inbound
cat src/proxy/inbound.rs | head -100

# Archivo clave para outbound
cat src/proxy/outbound.rs | head -100
```

### Ver xDS Client

```bash
# Cómo se conecta a istiod
cat src/xds/mod.rs | head -100
```

---

## 12. Siguiente Paso

Una vez completado, continúa con:
- [02_analizar_conexion.md](02_analizar_conexion.md) - Analizar Flujo de Conexión

---

**Tiempo total estimado**: 1-2 horas

