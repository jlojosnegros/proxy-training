# Ejercicio: Analizar Flujo de Conexión en ztunnel

---

**Tipo**: Ejercicio práctico
**Tiempo estimado**: 2 horas
**Prerrequisitos**: Ejercicio 01 completado

---

## Objetivo

Entender cómo ztunnel procesa una conexión siguiendo el código fuente.

---

## 1. Preparación

### 1.1 Abrir el Repositorio

```bash
cd /home/jojosneg/source/redhat/ztunnel/upstream/master
```

### 1.2 Tener IDE Listo

Recomendado: VS Code con rust-analyzer

```bash
# Instalar rust-analyzer si no lo tienes
rustup component add rust-analyzer
```

---

## 2. Trazar Inbound Connection

### 2.1 Entry Point de Inbound

```bash
# Buscar donde se aceptan conexiones inbound
grep -r "accept" src/proxy/inbound.rs | head -20
```

Busca una función similar a:

```rust
// Pseudocódigo basado en estructura típica
async fn accept_inbound(listener: TcpListener) {
    loop {
        let (stream, addr) = listener.accept().await?;
        tokio::spawn(handle_inbound(stream, addr));
    }
}
```

### 2.2 Handler de Conexión

Encuentra el handler principal:

```bash
# Buscar función de handle
grep -rn "async fn.*handle\|async fn.*process" src/proxy/inbound.rs
```

Analiza la secuencia:

```rust
async fn handle_inbound(stream: TcpStream) -> Result<()> {
    // 1. ¿Cómo obtiene el destino original?
    let original_dst = get_original_dst(&stream)?;

    // 2. ¿Cómo busca información del workload?
    let workload = state.workloads.get(&original_dst)?;

    // 3. ¿Cómo establece la conexión al pod?
    let upstream = connect_to_workload(&workload).await?;

    // 4. ¿Cómo hace el proxy bidireccional?
    proxy(stream, upstream).await
}
```

---

## 3. Trazar Outbound Connection

### 3.1 Entry Point de Outbound

```bash
# Buscar funciones de outbound
grep -rn "outbound\|15001" src/proxy/
```

### 3.2 Flujo de Outbound

```rust
async fn handle_outbound(stream: TcpStream) -> Result<()> {
    // 1. Obtener destino original (SO_ORIGINAL_DST)
    let original_dst = get_original_dst(&stream)?;

    // 2. Buscar en xDS cache
    let dest_info = xds_cache.lookup(&original_dst)?;

    // 3. Determinar si destino está en mesh
    if dest_info.in_mesh {
        // Usar HBONE
        connect_hbone(dest_info).await
    } else {
        // Passthrough directo
        connect_direct(original_dst).await
    }
}
```

---

## 4. Analizar HBONE

### 4.1 Encontrar Código HBONE

```bash
# Buscar implementación de HBONE
grep -rn "HBONE\|hbone\|CONNECT" src/
ls src/*hbone* 2>/dev/null || find src -name "*hbone*"
```

### 4.2 Entender el Flujo

```rust
async fn establish_hbone(
    dest: SocketAddr,
    source_identity: &Identity,
) -> Result<Stream> {
    // 1. Conectar TCP al ztunnel remoto
    let tcp = TcpStream::connect(remote_ztunnel).await?;

    // 2. TLS handshake con cert del workload source
    let tls = do_mtls(tcp, source_identity).await?;

    // 3. HTTP/2 CONNECT request
    let h2 = h2::client::handshake(tls).await?;
    let (send, recv) = h2.ready().await?;

    // 4. Enviar CONNECT
    let req = Request::builder()
        .method(Method::CONNECT)
        .uri(format!("https://{}/", dest))
        .body(())?;

    let response = send.send_request(req, false)?.await?;

    // 5. Retornar stream
    Ok(Stream::new(send, recv))
}
```

---

## 5. Analizar Manejo de Certificados

### 5.1 Encontrar Identity Module

```bash
# Ver estructura de identity
ls src/identity/
cat src/identity/mod.rs | head -50
```

### 5.2 Trazar Obtención de Cert

```rust
// Buscar cómo obtiene certificados para un workload
async fn get_certificate(workload: &Workload) -> Result<Certificate> {
    // 1. Verificar cache
    if let Some(cert) = cache.get(&workload.spiffe_id) {
        if !cert.is_expired() {
            return Ok(cert);
        }
    }

    // 2. Solicitar a istiod via SDS
    let csr = create_csr(&workload.spiffe_id)?;
    let cert = sds_client.request_cert(csr).await?;

    // 3. Cachear
    cache.insert(workload.spiffe_id.clone(), cert.clone());

    Ok(cert)
}
```

---

## 6. Añadir Logging para Debugging

### 6.1 Modificar Código

Encuentra un archivo apropiado y añade logging:

```rust
// En src/proxy/inbound.rs o similar

use tracing::{info, debug, error};

async fn handle_connection(stream: TcpStream) -> Result<()> {
    info!("New connection from {:?}", stream.peer_addr());

    let original_dst = get_original_dst(&stream)?;
    debug!("Original destination: {:?}", original_dst);

    // ... resto del código ...

    info!("Connection completed");
    Ok(())
}
```

### 6.2 Compilar y Probar

```bash
cargo build

RUST_LOG=debug ./target/debug/ztunnel
```

---

## 7. Ejercicio: Seguir un Request

### 7.1 Diagrama de Secuencia

Dibuja el flujo basándote en tu análisis:

```
Client App
    │
    │ connect(10.0.2.5:8080)
    ▼
iptables (REDIRECT to 15001)
    │
    ▼
ztunnel A:15001 (outbound)
    │
    │ 1. read SO_ORIGINAL_DST → 10.0.2.5:8080
    │ 2. lookup workload info
    │ 3. get source cert
    │ 4. connect to ztunnel B:15008
    │
    ▼
ztunnel B:15008 (HBONE inbound)
    │
    │ 1. TLS handshake (verify source cert)
    │ 2. receive CONNECT request
    │ 3. connect to local pod
    │
    ▼
Pod B:8080
```

### 7.2 Identificar en Código

Para cada paso, encuentra el archivo y función correspondiente:

| Paso             | Archivo                 | Función/Línea      |
| ---------------- | ----------------------- | ------------------ |
| Accept outbound  | `src/proxy/outbound.rs` | `accept_*`         |
| Get original dst | `src/proxy/???`         | `get_original_dst` |
| Lookup workload  | `src/xds/???`           | `lookup`           |
| Get cert         | `src/identity/???`      | `get_cert`         |
| HBONE connect    | `src/proxy/hbone.rs?`   | `connect_*`        |

---

## 8. Preguntas de Comprensión

Responde basándote en tu análisis:

1. **¿Dónde se decide si usar HBONE o passthrough directo?**

2. **¿Cómo sabe ztunnel qué certificado usar para outbound?**

3. **¿Qué pasa si el workload destino no está en el xDS cache?**

4. **¿Cómo se propaga el error si la conexión upstream falla?**

5. **¿Dónde se aplican las AuthorizationPolicies?**

---

## 9. Checkpoints

- [ ] Entry point de inbound identificado
- [ ] Entry point de outbound identificado
- [ ] Flujo HBONE entendido
- [ ] Manejo de certificados trazado
- [ ] Diagrama de secuencia completado
- [ ] Preguntas respondidas

---

## 10. Recursos Adicionales

```bash
# Ver todos los logs posibles
grep -r "info!\|debug!\|error!\|trace!" src/ | wc -l

# Ver uso de Tokio select
grep -r "select!" src/

# Ver uso de channels
grep -r "mpsc\|oneshot\|broadcast" src/
```

---

## 11. Conclusión

Después de este ejercicio deberías:

1. **Entender el flujo de conexión** desde accept hasta proxy
2. **Conocer los módulos principales** y su responsabilidad
3. **Saber dónde buscar** para debugging
4. **Comprender la interacción** entre proxy, xDS e identity

---

**Plan de formación completado!**

Para continuar aprendiendo:

- Contribuye un fix o feature pequeño
- Añade tests para casos edge
- Documenta algo que encontraste confuso
- Experimenta con la configuración
