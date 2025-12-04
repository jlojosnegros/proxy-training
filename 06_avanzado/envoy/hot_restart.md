# Hot Restart en Envoy

---

**Módulo**: 6 - Avanzado (Envoy)
**Tema**: Mecanismo de Hot Restart
**Tiempo estimado**: 2 horas
**Prerrequisitos**: Módulo 3 completo

---

## Objetivos de Aprendizaje

Al completar este documento:

- Entenderás cómo funciona hot restart en Envoy
- Conocerás el proceso de transferencia de estado
- Comprenderás las limitaciones y trade-offs
- Sabrás cuándo y cómo usar hot restart

---

## 1. ¿Qué es Hot Restart?

### 1.1 El Problema

```
┌─────────────────────────────────────────────────────────────────┐
│               El Problema de Upgrade/Restart                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sin Hot Restart:                                              │
│                                                                 │
│  1. Envoy v1 running ──────────────────────────────           │
│                                                                 │
│  2. Kill v1           ╔══════════════════════════╗            │
│     ──────────────────║     DOWNTIME             ║            │
│                       ║  Conexiones perdidas     ║            │
│                       ║  Requests fallidos       ║            │
│                       ╚══════════════════════════╝            │
│                                                                 │
│  3. Start v2          ──────────────────────────────           │
│                                                                 │
│  Problema: Durante el restart:                                 │
│  • Conexiones activas se pierden                               │
│  • Nuevas conexiones fallan                                    │
│  • Requests in-flight se pierden                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 La Solución: Hot Restart

```
┌─────────────────────────────────────────────────────────────────┐
│                      Hot Restart                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Envoy v1 running ──────────────────────────────────────    │
│                                                                 │
│  2. Start v2 (parent)                                          │
│     v1 sigue activo  ──────────────────────────────            │
│     v2 inicia        ──────────────────────────────            │
│                                                                 │
│  3. Drain period                                               │
│     v1: draining     ──────────────────────────────            │
│         (no new connections)                                    │
│     v2: accepting    ──────────────────────────────            │
│         (new connections)                                       │
│                                                                 │
│  4. v1 connections finish                                      │
│     v1: terminates   ──────                                    │
│     v2: fully active ──────────────────────────────            │
│                                                                 │
│  Resultado:                                                    │
│  • Zero downtime                                               │
│  • Conexiones existentes terminan gracefully                   │
│  • Nuevas conexiones van al nuevo proceso                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Arquitectura de Hot Restart

### 2.1 Componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                  Hot Restart Architecture                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Parent Process                          │ │
│  │                    (Envoy v2 - nuevo)                      │ │
│  │                                                            │ │
│  │  ┌─────────────┐   ┌─────────────────────────────────┐   │ │
│  │  │Hot Restart  │   │        Workers                   │   │ │
│  │  │  Manager    │   │  (accepting new connections)     │   │ │
│  │  └──────┬──────┘   └─────────────────────────────────┘   │ │
│  │         │                                                  │ │
│  └─────────┼──────────────────────────────────────────────────┘ │
│            │                                                     │
│            │ Unix Domain Socket                                  │
│            │ (shared memory / fd passing)                        │
│            │                                                     │
│  ┌─────────┼──────────────────────────────────────────────────┐ │
│  │         ▼                                                   │ │
│  │  ┌──────────────┐   ┌─────────────────────────────────┐   │ │
│  │  │Hot Restart   │   │        Workers                   │   │ │
│  │  │  Manager     │   │  (draining old connections)      │   │ │
│  │  └──────────────┘   └─────────────────────────────────┘   │ │
│  │                                                            │ │
│  │                    Child Process                           │ │
│  │                    (Envoy v1 - viejo)                      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Qué se Transfiere

| Datos                  | Se transfiere | Cómo                          |
| ---------------------- | ------------- | ----------------------------- |
| **Listen sockets**     | ✓ Sí          | Unix domain socket FD passing |
| **Stats**              | ✓ Sí          | Shared memory                 |
| **Active connections** | ✗ No          | Drain naturalmente            |
| **Connection pools**   | ✗ No          | Se recrean                    |
| **Config**             | ✗ No          | Nuevo proceso lee config      |
| **Certificates**       | ✗ No          | Nuevo proceso carga certs     |

---

## 3. Proceso de Hot Restart

### 3.1 Secuencia Detallada

```
┌─────────────────────────────────────────────────────────────────┐
│                   Hot Restart Sequence                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T=0: Envoy v1 running                                         │
│  ───────────────────────────────────────────────────────────   │
│  v1: Listening on ports 80, 443                                │
│      Active connections: 1000                                  │
│                                                                 │
│  T=1: Start Envoy v2 with --restart-epoch 1                    │
│  ───────────────────────────────────────────────────────────   │
│  v2: Connects to v1 via hot restart socket                     │
│  v2: Requests listen socket FDs from v1                        │
│  v1: Passes socket FDs via SCM_RIGHTS                          │
│  v2: Binds to same ports (using passed FDs)                    │
│                                                                 │
│  T=2: v2 starts accepting connections                          │
│  ───────────────────────────────────────────────────────────   │
│  v2: Sends "drain" signal to v1                                │
│  v1: Starts drain sequence                                     │
│      - Stops accepting new connections                         │
│      - Existing connections continue                           │
│                                                                 │
│  T=3: Drain period (configurable, default varies)              │
│  ───────────────────────────────────────────────────────────   │
│  v1: Connections finishing naturally                           │
│      Active: 1000 → 500 → 100 → 0                             │
│  v2: Handling all new connections                              │
│                                                                 │
│  T=4: Drain complete                                           │
│  ───────────────────────────────────────────────────────────   │
│  v1: All connections closed                                    │
│  v1: Terminates                                                │
│  v2: Now sole running instance                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Código Relevante

```cpp
// source/server/hot_restart_impl.h

class HotRestartImpl : public HotRestart {
public:
    // Inicializa hot restart
    HotRestartImpl(Options& options);

    // Notifica a viejo proceso que drene
    void sendParentAdminShutdownRequest() override;

    // Pasa socket FDs
    Network::ListenSocketSharedPtr duplicateListenSocket(
        const std::string& address,
        uint32_t worker_index) override;

    // Transfiere stats
    void exportStatsToParent(Stats::StatNameHashSet& stat_names) override;

private:
    // Unix domain socket para comunicación
    int my_domain_socket_;
    int parent_domain_socket_;

    // Shared memory para stats
    SharedMemory& shmem_;
};
```

---

## 4. Configuración

### 4.1 Opciones de Línea de Comandos

```bash
# Primer inicio (epoch 0)
envoy -c envoy.yaml --restart-epoch 0

# Segundo inicio (hot restart)
envoy -c envoy.yaml --restart-epoch 1

# Con drain time personalizado
envoy -c envoy.yaml --restart-epoch 1 \
    --drain-time-s 30 \
    --parent-shutdown-time-s 60
```

### 4.2 Parámetros Importantes

| Parámetro                  | Default | Descripción                                |
| -------------------------- | ------- | ------------------------------------------ |
| `--restart-epoch`          | 0       | Epoch actual (incrementar en cada restart) |
| `--drain-time-s`           | 600     | Tiempo de drain en segundos                |
| `--parent-shutdown-time-s` | 900     | Tiempo máximo para shutdown del parent     |
| `--hot-restart-version`    | -       | Muestra versión del protocolo              |
| `--base-id`                | 0       | ID base para shared memory                 |

### 4.3 Wrapper Script

```bash
#!/bin/bash
# hot_restart.sh

ENVOY_BIN="/usr/local/bin/envoy"
CONFIG="/etc/envoy/envoy.yaml"
EPOCH_FILE="/var/run/envoy/epoch"

# Leer epoch actual
if [ -f "$EPOCH_FILE" ]; then
    CURRENT_EPOCH=$(cat "$EPOCH_FILE")
else
    CURRENT_EPOCH=-1
fi

# Incrementar epoch
NEW_EPOCH=$((CURRENT_EPOCH + 1))
echo $NEW_EPOCH > "$EPOCH_FILE"

# Iniciar Envoy con nuevo epoch
exec $ENVOY_BIN -c $CONFIG \
    --restart-epoch $NEW_EPOCH \
    --drain-time-s 30 \
    --parent-shutdown-time-s 60
```

---

## 5. Stats y Shared Memory

### 5.1 Transferencia de Stats

```
┌─────────────────────────────────────────────────────────────────┐
│                  Stats Hot Restart                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Shared Memory Layout:                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Slot 0 (epoch 0)    Slot 1 (epoch 1)                   │   │
│  │  ┌─────────────┐     ┌─────────────┐                    │   │
│  │  │ Counter: 100│     │ Counter: 150│                    │   │
│  │  │ Gauge: 50   │     │ Gauge: 45   │                    │   │
│  │  │ ...         │     │ ...         │                    │   │
│  │  └─────────────┘     └─────────────┘                    │   │
│  │                                                          │   │
│  │  Durante hot restart:                                   │   │
│  │  v2 lee stats de v1 slot                                │   │
│  │  v2 continúa contadores desde valor existente           │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Resultado:                                                    │
│  • Counters son monotónicos a través de restarts             │
│  • Histogramas se resetean                                    │
│  • Gauges se transfieren                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Configurar Shared Memory

```bash
# Ver shared memory segments
ls -la /dev/shm/envoy*

# Calcular tamaño necesario
# Depende del número de stats
# Default: ~10MB

# Si necesitas más
envoy -c envoy.yaml \
    --max-obj-name-len 500 \
    --max-stats 50000
```

---

## 6. Draining

### 6.1 Comportamiento de Drain

```cpp
// source/server/drain_manager_impl.cc

class DrainManagerImpl : public DrainManager {
    void startDrainSequence(std::function<void()> completion) override {
        // Notificar a listeners que drenen
        for (auto& listener : listeners_) {
            listener->startDrain();
        }

        // Timer para completar drain
        drain_timer_ = dispatcher_.createTimer([this, completion]() {
            if (draining_ && checkDrainComplete()) {
                completion();
            }
        });

        drain_timer_->enableTimer(drain_time_);
    }

    bool checkDrainComplete() {
        // Verificar que todas las conexiones cerraron
        for (auto& listener : listeners_) {
            if (listener->activeConnections() > 0) {
                return false;
            }
        }
        return true;
    }
};
```

### 6.2 Tipos de Drain

```
┌─────────────────────────────────────────────────────────────────┐
│                      Drain Types                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. GRADUAL DRAIN (default)                                    │
│  ────────────────────────────                                   │
│  • Conexiones existentes continúan                             │
│  • Nuevas conexiones rechazadas                                │
│  • Ideal para HTTP/1.1 y corta duración                        │
│                                                                 │
│  2. IMMEDIATE DRAIN                                            │
│  ───────────────────────                                        │
│  • Cierra conexiones idle inmediatamente                       │
│  • Conexiones con requests in-flight esperan                   │
│  • Para casos donde drain rápido es necesario                  │
│                                                                 │
│  3. LB DRAIN                                                   │
│  ─────────────                                                  │
│  • Solo para endpoints, no para Envoy                          │
│  • Envía health check unhealthy                                │
│  • LB deja de enviar tráfico                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Limitaciones

### 7.1 Qué NO se Transfiere

```
┌─────────────────────────────────────────────────────────────────┐
│                     NOT Transferred                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ACTIVE CONNECTIONS                                         │
│     ───────────────────                                         │
│     • HTTP/2 streams en progreso                               │
│     • WebSocket connections                                     │
│     • Long-polling connections                                  │
│     → Deben terminar naturalmente durante drain                │
│                                                                 │
│  2. CONNECTION POOLS                                           │
│     ──────────────────                                          │
│     • Upstream connections                                      │
│     • Deben recrearse en nuevo proceso                         │
│     → Brief spike en latencia al start                         │
│                                                                 │
│  3. CACHED DATA                                                │
│     ───────────────                                             │
│     • Route caches                                             │
│     • DNS caches                                               │
│     • Certificate caches                                       │
│     → Se reconstruyen al start                                 │
│                                                                 │
│  4. xDS STATE                                                  │
│     ──────────                                                  │
│     • Nuevo proceso hace nueva subscripción                    │
│     • Puede haber brief inconsistency                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 Requisitos del Sistema

```bash
# Hot restart requiere:

# 1. Mismo usuario (para shared memory)
# 2. Mismo base-id
# 3. Epoch consecutivo

# Verificar soporte
envoy --hot-restart-version

# Típicamente:
# 11.104
# ^^  ^^^
# |   |
# |   └── Minor version
# └────── Major version
```

---

## 8. Alternativas

### 8.1 Comparación

| Método           | Downtime | Complejidad | Use Case              |
| ---------------- | -------- | ----------- | --------------------- |
| **Hot Restart**  | Zero     | Media       | Config/binary updates |
| **xDS**          | Zero     | Baja        | Config updates only   |
| **Kill + Start** | Segundos | Baja        | Non-critical          |
| **Blue/Green**   | Zero     | Alta        | Major changes         |

### 8.2 Cuándo Usar Cada Uno

```
┌─────────────────────────────────────────────────────────────────┐
│                  Choosing Update Method                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ¿Qué estás actualizando?                                      │
│  │                                                              │
│  ├── Solo configuración                                        │
│  │   └── Usa xDS si posible (zero downtime, más simple)       │
│  │                                                              │
│  ├── Binary de Envoy                                           │
│  │   └── Usa Hot Restart                                       │
│  │                                                              │
│  ├── Configuración bootstrap (no xDS)                          │
│  │   └── Usa Hot Restart                                       │
│  │                                                              │
│  └── Cambios mayores (ports, etc)                              │
│      └── Blue/Green deployment                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Autoevaluación

1. ¿Qué problema resuelve hot restart?
2. ¿Qué se transfiere entre procesos durante hot restart?
3. ¿Qué pasa con las conexiones activas durante hot restart?
4. ¿Qué es el "drain period"?
5. ¿Cuándo usarías xDS en lugar de hot restart?

---

## 10. Referencias

| Recurso                                                                                                           | Descripción           |
| ----------------------------------------------------------------------------------------------------------------- | --------------------- |
| [Hot Restart Blog](https://blog.envoyproxy.io/envoy-hot-restart-1d16b14555b5)                                     | Explicación detallada |
| `source/server/hot_restart_impl.cc`                                                                               | Implementación        |
| `source/server/drain_manager_impl.cc`                                                                             | Drain logic           |
| [Envoy Docs: Hot Restart](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/hot_restart) | Documentación oficial |

---

**Siguiente**: [performance_tuning.md](performance_tuning.md) - Performance Tuning
