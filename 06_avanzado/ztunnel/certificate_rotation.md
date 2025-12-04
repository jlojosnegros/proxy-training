# Rotación de Certificados SPIFFE en ztunnel

---

**Módulo**: 6 - Avanzado (ztunnel)
**Tema**: Gestión y rotación de certificados
**Tiempo estimado**: 2 horas
**Prerrequisitos**: Módulo 4 completo

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás cómo ztunnel gestiona certificados
- Conocerás el flujo de rotación de certificados
- Comprenderás la integración con istiod CA
- Sabrás debugging de problemas de certificados

---

## 1. Certificados en ztunnel

### 1.1 Tipos de Certificados

```
┌─────────────────────────────────────────────────────────────────┐
│                    Certificates in ztunnel                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. WORKLOAD CERTIFICATES (SVID)                               │
│  ────────────────────────────────                               │
│  • Un certificado por cada workload en el nodo                 │
│  • Identidad SPIFFE: spiffe://cluster.local/ns/X/sa/Y          │
│  • Usado para mTLS entre workloads                             │
│  • Vida corta: 24 horas típicamente                            │
│                                                                 │
│  2. CA ROOT CERTIFICATE                                        │
│  ───────────────────────                                        │
│  • Certificado raíz de la CA de Istio                          │
│  • Usado para validar certificados de otros workloads          │
│  • Típicamente vida larga                                      │
│                                                                 │
│  3. ZTUNNEL'S OWN CERTIFICATE                                  │
│  ─────────────────────────────                                  │
│  • Certificado del propio ztunnel                              │
│  • Usado para admin/metrics endpoints                          │
│  • NO usado para mTLS de workloads                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 SPIFFE SVID Format

```
┌─────────────────────────────────────────────────────────────────┐
│                         SPIFFE SVID                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  X.509 Certificate:                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Subject: O=cluster.local                               │   │
│  │                                                          │   │
│  │  Subject Alternative Names (SAN):                        │   │
│  │    URI: spiffe://cluster.local/ns/default/sa/frontend   │   │
│  │                     │           │          │             │   │
│  │                     │           │          └─ Service Account│
│  │                     │           └──────────── Namespace  │   │
│  │                     └──────────────────────── Trust Domain│  │
│  │                                                          │   │
│  │  Issuer: spiffe://cluster.local                         │   │
│  │                                                          │   │
│  │  Validity:                                               │   │
│  │    Not Before: 2025-12-04T10:00:00Z                     │   │
│  │    Not After:  2025-12-05T10:00:00Z  (24 hours)         │   │
│  │                                                          │   │
│  │  Key Usage:                                              │   │
│  │    Digital Signature                                     │   │
│  │    Key Encipherment                                      │   │
│  │                                                          │   │
│  │  Extended Key Usage:                                     │   │
│  │    TLS Web Server Authentication                        │   │
│  │    TLS Web Client Authentication                        │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Arquitectura de Gestión de Certificados

### 2.1 Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│               Certificate Management Architecture              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                        istiod                            │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │                   Citadel CA                        │ │   │
│  │  │                                                     │ │   │
│  │  │  • Signs CSRs from workloads                       │ │   │
│  │  │  • Validates workload identity                     │ │   │
│  │  │  • Issues short-lived certificates                 │ │   │
│  │  │                                                     │ │   │
│  │  └──────────────────────┬─────────────────────────────┘ │   │
│  └─────────────────────────┼───────────────────────────────┘   │
│                            │                                    │
│                            │ SDS (Secret Discovery Service)    │
│                            │ gRPC stream                        │
│                            │                                    │
│  ┌─────────────────────────┼───────────────────────────────┐   │
│  │         ztunnel         │                                │   │
│  │  ┌──────────────────────▼─────────────────────────────┐ │   │
│  │  │              Certificate Manager                    │ │   │
│  │  │                                                     │ │   │
│  │  │  • Maintains certs for all workloads on node       │ │   │
│  │  │  • Monitors expiration                             │ │   │
│  │  │  • Triggers rotation before expiry                 │ │   │
│  │  │                                                     │ │   │
│  │  └──────────────────────┬─────────────────────────────┘ │   │
│  │                         │                                │   │
│  │  ┌──────────────────────┼─────────────────────────────┐ │   │
│  │  │      Certificate Store                              │ │   │
│  │  │  ┌─────────────────────────────────────────────┐   │ │   │
│  │  │  │ Workload A → Cert A + Key A                 │   │ │   │
│  │  │  │ Workload B → Cert B + Key B                 │   │ │   │
│  │  │  │ Workload C → Cert C + Key C                 │   │ │   │
│  │  │  │ CA Root   → Root Cert                       │   │ │   │
│  │  │  └─────────────────────────────────────────────┘   │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Flujo de Obtención de Certificado

```
┌─────────────────────────────────────────────────────────────────┐
│              Certificate Request Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. New workload starts on node                                │
│     Pod scheduled, ztunnel notices via xDS                     │
│                                                                 │
│  2. ztunnel generates key pair                                 │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ let keypair = generate_ec_p256_keypair();           │    │
│     │ let csr = create_csr(&keypair, &spiffe_id);         │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. ztunnel sends CSR to istiod via SDS                        │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ SDS Request:                                         │    │
│     │   resource_name: "spiffe://cluster.local/ns/X/sa/Y" │    │
│     │   csr: <base64 encoded CSR>                         │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  4. istiod validates and signs                                 │
│     • Verifies ztunnel is on correct node                      │
│     • Verifies workload exists                                 │
│     • Signs CSR with CA key                                    │
│                                                                 │
│  5. istiod returns signed certificate                          │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ SDS Response:                                        │    │
│     │   certificate_chain: [cert, intermediate, root]     │    │
│     │   valid_until: <timestamp>                          │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  6. ztunnel stores cert for workload                           │
│     Ready to use for mTLS connections                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Rotación de Certificados

### 3.1 Por qué Rotar

```
┌─────────────────────────────────────────────────────────────────┐
│                 Why Certificate Rotation                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. SECURITY                                                   │
│     • Limita ventana de compromiso                             │
│     • Si clave comprometida, solo válida 24h                   │
│     • Cumple con zero trust principles                         │
│                                                                 │
│  2. REVOCATION IS HARD                                         │
│     • OCSP/CRL son lentos y complicados                        │
│     • Vida corta = rotación natural                            │
│                                                                 │
│  3. KEY ROTATION                                               │
│     • Nuevas claves regularmente                               │
│     • Mitiga cryptographic attacks                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Timing de Rotación

```
┌─────────────────────────────────────────────────────────────────┐
│                    Rotation Timeline                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Certificate Lifetime: 24 hours                                │
│                                                                 │
│  ├────────────────────────────────────────────────────────────┤│
│  │                                                             ││
│  │  0h        6h        12h       18h       24h               ││
│  │  │         │         │         │         │                 ││
│  │  ▼         ▼         ▼         ▼         ▼                 ││
│  │  ┌─────────────────────────────────────────────────────┐  ││
│  │  │████████████████████████████████████████████████████│  ││
│  │  │             Certificate Valid                       │  ││
│  │  │                                                     │  ││
│  │  │                          ┌──────────────────────────│  ││
│  │  │                          │  Rotation Window         │  ││
│  │  │                          │  (start at 50% lifetime) │  ││
│  │  │                          └──────────────────────────│  ││
│  │  └─────────────────────────────────────────────────────┘  ││
│  │                                                             ││
│  │  At 12h (50% of lifetime):                                 ││
│  │  • ztunnel starts rotation                                 ││
│  │  • Requests new certificate from istiod                    ││
│  │  • Old cert still valid                                    ││
│  │                                                             ││
│  │  Before 24h:                                               ││
│  │  • New cert in use                                         ││
│  │  • Old cert discarded                                      ││
│  │                                                             ││
│  ├────────────────────────────────────────────────────────────┤│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Pseudocódigo de Rotación

```rust
// Simplified certificate rotation logic

struct CertificateManager {
    certs: HashMap<SpiffeId, CertificateState>,
    sds_client: SdsClient,
}

struct CertificateState {
    certificate: X509,
    private_key: PrivateKey,
    expiry: Instant,
    rotation_scheduled: bool,
}

impl CertificateManager {
    async fn monitor_certificates(&self) {
        let mut interval = tokio::time::interval(Duration::from_secs(60));

        loop {
            interval.tick().await;

            for (id, state) in &self.certs {
                let remaining = state.expiry - Instant::now();
                let lifetime = state.certificate.lifetime();

                // Rotate at 50% of lifetime
                if remaining < lifetime / 2 && !state.rotation_scheduled {
                    self.schedule_rotation(id.clone()).await;
                }
            }
        }
    }

    async fn schedule_rotation(&self, id: SpiffeId) {
        let new_cert = self.request_certificate(&id).await?;

        // Atomic swap
        self.certs.insert(id, CertificateState {
            certificate: new_cert.cert,
            private_key: new_cert.key,
            expiry: new_cert.expiry,
            rotation_scheduled: false,
        });

        info!("Rotated certificate for {}", id);
    }

    async fn request_certificate(&self, id: &SpiffeId) -> Result<Certificate> {
        // Generate new key pair
        let keypair = generate_keypair()?;

        // Create CSR
        let csr = create_csr(&keypair, id)?;

        // Request from istiod via SDS
        let response = self.sds_client.request(id, &csr).await?;

        Ok(Certificate {
            cert: response.certificate_chain,
            key: keypair.private_key,
            expiry: response.valid_until,
        })
    }
}
```

---

## 4. Uso de Certificados

### 4.1 En Conexiones Outbound

```rust
async fn establish_mtls_outbound(
    source_workload: &Workload,
    dest_addr: SocketAddr,
    cert_manager: &CertificateManager,
) -> Result<TlsStream> {
    // 1. Obtener certificado del source workload
    let source_id = &source_workload.spiffe_id;
    let cert_state = cert_manager.get_cert(source_id)?;

    // 2. Configurar TLS con el certificado del workload
    let tls_config = TlsClientConfig::builder()
        .with_certificate(cert_state.certificate.clone())
        .with_private_key(cert_state.private_key.clone())
        .with_root_cas(cert_manager.root_cas())
        .build()?;

    // 3. Conectar con mTLS
    let tcp_stream = TcpStream::connect(dest_addr).await?;
    let tls_stream = tls_config.connect(tcp_stream).await?;

    // 4. Verificar identidad del server
    let server_cert = tls_stream.peer_certificate()?;
    let server_id = extract_spiffe_id(&server_cert)?;

    // Aquí se podrían aplicar authorization policies
    verify_allowed(source_id, &server_id)?;

    Ok(tls_stream)
}
```

### 4.2 En Conexiones Inbound

```rust
async fn accept_mtls_inbound(
    listener: TlsListener,
    cert_manager: &CertificateManager,
) -> Result<()> {
    loop {
        let (tls_stream, _addr) = listener.accept().await?;

        // 1. Verificar certificado del client
        let client_cert = tls_stream.peer_certificate()?;
        let client_id = extract_spiffe_id(&client_cert)?;

        // 2. Determinar workload destino
        let original_dst = get_original_dst(&tls_stream)?;
        let dest_workload = lookup_workload(&original_dst)?;

        // 3. Verificar authorization
        verify_authorization(&client_id, &dest_workload)?;

        // 4. Procesar conexión
        spawn(handle_connection(tls_stream, dest_workload));
    }
}
```

---

## 5. Debugging de Certificados

### 5.1 Ver Certificados Actuales

```bash
# Ver certificados que ztunnel tiene cargados
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=ztunnel -o name | head -1) -- \
    curl localhost:15000/debug/certs

# Output ejemplo:
# {
#   "certificates": [
#     {
#       "spiffe_id": "spiffe://cluster.local/ns/default/sa/frontend",
#       "valid_from": "2025-12-04T10:00:00Z",
#       "valid_until": "2025-12-05T10:00:00Z",
#       "serial_number": "abc123..."
#     }
#   ]
# }
```

### 5.2 Verificar Rotación

```bash
# Logs de rotación
kubectl logs -n istio-system -l app=ztunnel | grep -i "certificate\|rotation\|sds"

# Ejemplo de log de rotación exitosa:
# INFO  ztunnel::identity: Certificate rotation started for spiffe://...
# INFO  ztunnel::identity: Certificate rotation completed for spiffe://...

# Ejemplo de error:
# ERROR ztunnel::identity: Certificate rotation failed: connection refused
```

### 5.3 Verificar Conectividad con istiod

```bash
# Ver conexión SDS
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=ztunnel -o name | head -1) -- \
    curl localhost:15000/debug/connections | jq '.sds'

# Verificar que istiod está accesible
kubectl exec -n istio-system $(kubectl get pod -n istio-system -l app=ztunnel -o name | head -1) -- \
    nc -zv istiod.istio-system.svc 15012
```

### 5.4 Problemas Comunes

```
┌─────────────────────────────────────────────────────────────────┐
│                  Common Certificate Issues                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. CERTIFICATE EXPIRED                                        │
│     Síntomas:                                                  │
│     • mTLS connections fail                                    │
│     • "certificate has expired" en logs                        │
│     Causas:                                                    │
│     • ztunnel no pudo conectar a istiod                       │
│     • istiod down o inaccesible                               │
│     • Clock skew entre nodos                                   │
│                                                                 │
│  2. SPIFFE ID MISMATCH                                         │
│     Síntomas:                                                  │
│     • "identity mismatch" errors                               │
│     • Authorization denied                                     │
│     Causas:                                                    │
│     • Service account cambió                                   │
│     • Namespace incorrecto                                     │
│                                                                 │
│  3. CA ROOT NOT TRUSTED                                        │
│     Síntomas:                                                  │
│     • "unknown CA" errors                                      │
│     • TLS handshake fails                                      │
│     Causas:                                                    │
│     • CA root rotó                                             │
│     • ztunnel no recibió nuevo root                           │
│                                                                 │
│  4. SDS CONNECTION FAILED                                      │
│     Síntomas:                                                  │
│     • No se obtienen certificados nuevos                       │
│     • "SDS connection error" en logs                          │
│     Causas:                                                    │
│     • istiod inaccesible                                       │
│     • Network policy bloqueando                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Configuración

### 6.1 Opciones de Rotación

```yaml
# Istio configuration for certificate rotation

# meshConfig en IstioOperator
spec:
  meshConfig:
    # Lifetime de certificados (default: 24h)
    certificates:
      - secretName: dns.example1-service-account
        dnsNames: [example1.default.svc, example1.default]

    # Trust domain
    trustDomain: cluster.local

    # Configuración de CA
    caCertificates:
      - pem: |
          -----BEGIN CERTIFICATE-----
          ...
          -----END CERTIFICATE-----
```

### 6.2 Variables de Entorno

```bash
# En ztunnel deployment

# Forzar rotación más frecuente (testing)
SECRET_TTL=3600  # 1 hour instead of 24h

# Endpoint de istiod
CA_ADDR=istiod.istio-system.svc:15012
```

---

## 7. Autoevaluación

1. ¿Por qué los certificados SPIFFE tienen vida corta?
2. ¿En qué punto del lifetime se inicia la rotación?
3. ¿Qué pasa si ztunnel no puede conectar con istiod durante rotación?
4. ¿De quién es el certificado que usa ztunnel para conexiones outbound?
5. ¿Cómo verificarías que la rotación está funcionando?

---

## 8. Referencias

| Recurso                                                                                   | Descripción              |
| ----------------------------------------------------------------------------------------- | ------------------------ |
| [SPIFFE Spec](https://spiffe.io/docs/latest/spiffe-about/overview/)                       | Especificación SPIFFE    |
| [Istio Security](https://istio.io/latest/docs/concepts/security/)                         | Seguridad en Istio       |
| ztunnel `src/identity/`                                                                   | Implementación           |
| [SDS Protocol](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret) | Secret Discovery Service |

---

**Siguiente**: [../../ejercicios/envoy/01_compilar_ejecutar.md](../../ejercicios/envoy/01_compilar_ejecutar.md) - Ejercicio: Compilar y Ejecutar Envoy
