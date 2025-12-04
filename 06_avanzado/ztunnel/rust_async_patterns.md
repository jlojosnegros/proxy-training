# Patrones Async en Rust para ztunnel

---

**Módulo**: 6 - Avanzado (ztunnel)
**Tema**: Patrones de programación async en Rust
**Tiempo estimado**: 3 horas
**Prerrequisitos**: Módulo 4 completo, conocimientos básicos de Rust

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás async/await en Rust
- Conocerás los patrones comunes en ztunnel
- Comprenderás Tokio runtime
- Podrás leer y modificar código async de ztunnel

---

## 1. Fundamentos de Async Rust

### 1.1 ¿Qué es Async?

```rust
// Async permite código concurrente sin múltiples threads

// Función SYNC (bloqueante)
fn read_file_sync(path: &str) -> String {
    std::fs::read_to_string(path).unwrap()
    // El thread se BLOQUEA esperando I/O
}

// Función ASYNC (no bloqueante)
async fn read_file_async(path: &str) -> String {
    tokio::fs::read_to_string(path).await.unwrap()
    // El thread puede hacer OTRAS cosas mientras espera
}
```

### 1.2 Futures y .await

```rust
// Un Future representa un valor que estará disponible "luego"

use std::future::Future;

// Esta función retorna un Future, no ejecuta nada todavía
async fn compute() -> i32 {
    42
}

// Para ejecutar el Future, usamos .await
async fn main() {
    let result = compute().await;  // Aquí se ejecuta
    println!("Result: {}", result);
}

// Internamente, compute() retorna algo como:
// impl Future<Output = i32>
```

### 1.3 El Runtime (Tokio)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Tokio Runtime                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Executor                              │   │
│  │  (decide qué Future ejecutar cuando)                    │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────┼────────────────────────────────┐   │
│  │                        │                                 │   │
│  │  ┌─────────────────────┴─────────────────────────────┐  │   │
│  │  │                Thread Pool                         │  │   │
│  │  │                                                    │  │   │
│  │  │  Thread 1      Thread 2      Thread 3             │  │   │
│  │  │  ┌───────┐     ┌───────┐     ┌───────┐           │  │   │
│  │  │  │Task A │     │Task D │     │Task G │           │  │   │
│  │  │  │Task B │     │Task E │     │Task H │           │  │   │
│  │  │  │Task C │     │Task F │     │       │           │  │   │
│  │  │  └───────┘     └───────┘     └───────┘           │  │   │
│  │  │                                                    │  │   │
│  │  │  Work-stealing: threads toman tareas de otros     │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │                                                          │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │                    Reactor                          │  │   │
│  │  │  (monitorea I/O events: epoll/kqueue)              │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Patrones Básicos

### 2.1 Spawn (Crear Tarea)

```rust
use tokio::spawn;

async fn main() {
    // Spawn crea una nueva tarea que corre concurrentemente
    let handle = spawn(async {
        // Esta tarea corre en el background
        println!("Hello from spawned task!");
        42
    });

    // Podemos esperar el resultado
    let result = handle.await.unwrap();
    println!("Result: {}", result);
}

// En ztunnel, spawn se usa para manejar conexiones:
async fn accept_connections(listener: TcpListener) {
    loop {
        let (stream, addr) = listener.accept().await.unwrap();

        // Cada conexión en su propia tarea
        spawn(async move {
            handle_connection(stream).await;
        });
    }
}
```

### 2.2 Select (Esperar Múltiples)

```rust
use tokio::select;

async fn main() {
    let future1 = async { 1 };
    let future2 = async { 2 };

    // select! espera CUALQUIERA de los futures
    select! {
        val = future1 => {
            println!("future1 completó primero: {}", val);
        }
        val = future2 => {
            println!("future2 completó primero: {}", val);
        }
    }
    // El otro future se CANCELA
}

// En ztunnel, para copiar datos bidireccionalmente:
async fn proxy_connection(
    mut client: TcpStream,
    mut server: TcpStream,
) -> Result<()> {
    let (mut client_read, mut client_write) = client.split();
    let (mut server_read, mut server_write) = server.split();

    select! {
        result = copy(&mut client_read, &mut server_write) => {
            result?;
        }
        result = copy(&mut server_read, &mut client_write) => {
            result?;
        }
    }

    Ok(())
}
```

### 2.3 Join (Esperar Todos)

```rust
use tokio::join;

async fn main() {
    let future1 = async { 1 };
    let future2 = async { 2 };

    // join! espera TODOS los futures
    let (result1, result2) = join!(future1, future2);

    println!("Results: {}, {}", result1, result2);
}

// Útil para inicialización paralela:
async fn initialize() {
    let (config, certs, xds) = join!(
        load_config(),
        load_certificates(),
        connect_to_xds(),
    );
}
```

---

## 3. Patrones en ztunnel

### 3.1 Connection Handler

```rust
// Patrón típico de ztunnel para manejar conexiones

pub async fn handle_inbound(
    downstream: TcpStream,
    state: Arc<State>,
) -> Result<()> {
    // 1. Obtener información del destino
    let original_dst = get_original_dst(&downstream)?;

    // 2. Buscar workload info
    let workload = state.workloads.get(&original_dst).await?;

    // 3. Establecer conexión upstream
    let upstream = connect_to_upstream(&workload).await?;

    // 4. Proxy bidireccional
    proxy_bidirectional(downstream, upstream).await
}

async fn proxy_bidirectional(
    downstream: TcpStream,
    upstream: TcpStream,
) -> Result<()> {
    let (down_read, down_write) = downstream.into_split();
    let (up_read, up_write) = upstream.into_split();

    let down_to_up = copy(down_read, up_write);
    let up_to_down = copy(up_read, down_write);

    // Esperar que cualquier dirección termine
    select! {
        result = down_to_up => result,
        result = up_to_down => result,
    }
}
```

### 3.2 Graceful Shutdown

```rust
use tokio::signal;
use tokio::sync::broadcast;

pub struct Server {
    shutdown_tx: broadcast::Sender<()>,
}

impl Server {
    pub async fn run(&self) -> Result<()> {
        let mut shutdown_rx = self.shutdown_tx.subscribe();

        loop {
            select! {
                // Aceptar nueva conexión
                result = self.listener.accept() => {
                    let (stream, addr) = result?;
                    let shutdown = self.shutdown_tx.subscribe();

                    spawn(async move {
                        handle_with_shutdown(stream, shutdown).await;
                    });
                }

                // Señal de shutdown
                _ = shutdown_rx.recv() => {
                    println!("Shutting down...");
                    break;
                }
            }
        }

        Ok(())
    }
}

async fn handle_with_shutdown(
    stream: TcpStream,
    mut shutdown: broadcast::Receiver<()>,
) {
    select! {
        _ = handle_connection(stream) => {}
        _ = shutdown.recv() => {
            // Cleanup on shutdown
        }
    }
}
```

### 3.3 State Sharing

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

// Estado compartido entre tareas
pub struct SharedState {
    workloads: RwLock<HashMap<String, Workload>>,
    certificates: RwLock<CertStore>,
}

impl SharedState {
    // Lectura - múltiples readers simultáneos
    pub async fn get_workload(&self, id: &str) -> Option<Workload> {
        let guard = self.workloads.read().await;
        guard.get(id).cloned()
    }

    // Escritura - exclusiva
    pub async fn update_workload(&self, id: String, workload: Workload) {
        let mut guard = self.workloads.write().await;
        guard.insert(id, workload);
    }
}

// Uso
async fn handler(state: Arc<SharedState>, request: Request) {
    let workload = state.get_workload(&request.target).await;
    // ... use workload
}
```

---

## 4. Channels en Tokio

### 4.1 mpsc (Multiple Producer, Single Consumer)

```rust
use tokio::sync::mpsc;

async fn main() {
    // Crear channel con buffer de 100
    let (tx, mut rx) = mpsc::channel(100);

    // Múltiples productores
    for i in 0..10 {
        let tx = tx.clone();
        spawn(async move {
            tx.send(i).await.unwrap();
        });
    }

    // Un consumidor
    drop(tx);  // Cerrar el sender original
    while let Some(value) = rx.recv().await {
        println!("Received: {}", value);
    }
}
```

### 4.2 watch (Single Value, Multiple Observers)

```rust
use tokio::sync::watch;

// Útil para config que cambia
async fn main() {
    let (tx, rx) = watch::channel(Config::default());

    // Watcher 1
    let mut rx1 = rx.clone();
    spawn(async move {
        while rx1.changed().await.is_ok() {
            let config = rx1.borrow();
            println!("Config updated: {:?}", *config);
        }
    });

    // Watcher 2
    let mut rx2 = rx.clone();
    spawn(async move {
        while rx2.changed().await.is_ok() {
            let config = rx2.borrow();
            apply_config(&*config);
        }
    });

    // Updater
    tx.send(new_config).unwrap();
}
```

### 4.3 oneshot (Single Value, Once)

```rust
use tokio::sync::oneshot;

// Útil para responses
async fn request_with_response() {
    let (tx, rx) = oneshot::channel();

    spawn(async move {
        // Hacer trabajo
        let result = expensive_computation().await;
        // Enviar resultado
        tx.send(result).unwrap();
    });

    // Esperar resultado
    let result = rx.await.unwrap();
}
```

---

## 5. Error Handling Async

### 5.1 Propagación con ?

```rust
async fn might_fail() -> Result<String, Error> {
    let data = read_file().await?;  // Propaga error si falla
    let parsed = parse(data)?;
    Ok(parsed)
}

// En ztunnel, típicamente:
async fn handle_connection(stream: TcpStream) -> Result<(), ProxyError> {
    let dst = get_original_dst(&stream).map_err(ProxyError::Socket)?;
    let workload = lookup_workload(&dst).await.map_err(ProxyError::Xds)?;
    let upstream = connect(&workload).await.map_err(ProxyError::Connect)?;

    proxy(stream, upstream).await.map_err(ProxyError::Proxy)
}
```

### 5.2 Timeouts

```rust
use tokio::time::{timeout, Duration};

async fn with_timeout() -> Result<(), Error> {
    // Timeout de 5 segundos
    match timeout(Duration::from_secs(5), slow_operation()).await {
        Ok(result) => result?,
        Err(_) => return Err(Error::Timeout),
    }

    Ok(())
}

// En ztunnel para connect timeout:
async fn connect_with_timeout(addr: SocketAddr) -> Result<TcpStream> {
    timeout(
        Duration::from_secs(10),
        TcpStream::connect(addr)
    )
    .await
    .map_err(|_| Error::ConnectTimeout)?
    .map_err(Error::Connect)
}
```

### 5.3 Retry Pattern

```rust
use tokio::time::sleep;

async fn with_retry<F, Fut, T, E>(
    mut operation: F,
    max_retries: u32,
    delay: Duration,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    let mut attempts = 0;

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempts < max_retries => {
                attempts += 1;
                sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}

// Uso
let result = with_retry(
    || connect_to_server(),
    3,
    Duration::from_secs(1),
).await?;
```

---

## 6. Patrones Avanzados

### 6.1 Stream Processing

```rust
use tokio_stream::{StreamExt, wrappers::ReceiverStream};

async fn process_stream() {
    let (tx, rx) = mpsc::channel(100);

    // Convertir a Stream
    let stream = ReceiverStream::new(rx);

    // Procesar con operadores de stream
    stream
        .filter(|x| x % 2 == 0)
        .map(|x| x * 2)
        .for_each(|x| async move {
            println!("Processed: {}", x);
        })
        .await;
}
```

### 6.2 Rate Limiting

```rust
use tokio::time::{interval, Duration};

struct RateLimiter {
    permits: Arc<Semaphore>,
}

impl RateLimiter {
    fn new(rate: u32) -> Self {
        let permits = Arc::new(Semaphore::new(rate as usize));

        // Reponer permits periódicamente
        let permits_clone = permits.clone();
        spawn(async move {
            let mut interval = interval(Duration::from_secs(1));
            loop {
                interval.tick().await;
                permits_clone.add_permits(rate as usize);
            }
        });

        Self { permits }
    }

    async fn acquire(&self) {
        self.permits.acquire().await.unwrap().forget();
    }
}
```

### 6.3 Connection Pool

```rust
use deadpool::managed::{Manager, Pool, RecycleResult};

struct TcpManager {
    addr: SocketAddr,
}

#[async_trait]
impl Manager for TcpManager {
    type Type = TcpStream;
    type Error = std::io::Error;

    async fn create(&self) -> Result<TcpStream, Self::Error> {
        TcpStream::connect(self.addr).await
    }

    async fn recycle(&self, conn: &mut TcpStream) -> RecycleResult<Self::Error> {
        // Verificar que la conexión sigue viva
        Ok(())
    }
}

// Uso
let pool = Pool::builder(TcpManager { addr })
    .max_size(100)
    .build()
    .unwrap();

let conn = pool.get().await?;
// Usar conexión
// Se devuelve al pool automáticamente al salir del scope
```

---

## 7. Debugging Async

### 7.1 Tokio Console

```toml
# Cargo.toml
[dependencies]
console-subscriber = "0.1"

[build]
rustflags = ["--cfg", "tokio_unstable"]
```

```rust
// main.rs
#[tokio::main]
async fn main() {
    console_subscriber::init();

    // ... resto del código
}
```

```bash
# Ejecutar tokio-console
tokio-console
```

### 7.2 Tracing

```rust
use tracing::{info, instrument};

#[instrument(skip(stream))]
async fn handle_connection(stream: TcpStream) -> Result<()> {
    info!("New connection");

    let result = process(&stream).await;

    info!("Connection closed");
    result
}
```

---

## 8. Autoevaluación

1. ¿Cuál es la diferencia entre `select!` y `join!`?
2. ¿Por qué usamos `Arc<RwLock<T>>` para estado compartido?
3. ¿Qué tipo de channel usarías para broadcasts de config updates?
4. ¿Cómo implementarías un timeout en una operación async?
5. ¿Qué pasa con el otro branch cuando un `select!` completa?

---

## 9. Referencias

| Recurso                                               | Descripción                        |
| ----------------------------------------------------- | ---------------------------------- |
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial)     | Tutorial oficial                   |
| [Async Book](https://rust-lang.github.io/async-book/) | Libro de async Rust                |
| [Tokio Console](https://github.com/tokio-rs/console)  | Debugging tool                     |
| ztunnel `src/`                                        | Código fuente para ejemplos reales |

---

**Siguiente**: [certificate_rotation.md](certificate_rotation.md) - Certificate Rotation
