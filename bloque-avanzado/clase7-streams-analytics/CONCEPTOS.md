# Conceptos: Kafka Streams y Patrones Event-Driven

Este documento explica los conceptos teóricos fundamentales de Kafka Streams, state stores, y el patrón CQRS.

---

## Tabla de contenidos

1. [¿Qué es Kafka Streams?](#qué-es-kafka-streams)
2. [Consumer Tradicional vs Kafka Streams](#consumer-tradicional-vs-kafka-streams)
3. [KStream vs KTable](#kstream-vs-ktable)
4. [State Stores](#state-stores)
5. [Operaciones Stateless vs Stateful](#operaciones-stateless-vs-stateful)
6. [Topología de Procesamiento](#topología-de-procesamiento)
7. [Patrón CQRS](#patrón-cqrs)
8. [Eventual Consistency](#eventual-consistency)
9. [Interactive Queries](#interactive-queries)

---

## ¿Qué es Kafka Streams?

**Kafka Streams** es una **biblioteca Java** para procesamiento de streams en tiempo real que:

- **Procesa datos directamente desde Kafka** sin cluster separado
- **Corre dentro de tu aplicación** Spring Boot (no requiere infraestructura adicional)
- **Es stateful** (puede mantener estado en memoria con state stores)
- **Es fault-tolerant** (estado respaldado en Kafka)
- **Escala horizontalmente** (agregar instancias para mayor throughput)

### Diferencias con otros frameworks

| Característica | Kafka Streams | Apache Flink | Spark Streaming |
|----------------|---------------|--------------|-----------------|
| Deployment | Biblioteca (en tu app) | Cluster separado | Cluster separado |
| Configuración | Mínima | Compleja | Moderada |
| Latencia | Milisegundos | Milisegundos | Segundos (micro-batch) |
| Caso de uso | Analytics simple, CQRS | Analytics complejo | Batch + streaming |

**Para este curso**: Kafka Streams es ideal porque no requiere infraestructura adicional y se integra naturalmente con Spring Boot.

---

## Consumer Tradicional vs Kafka Streams

### Consumer Tradicional (`@KafkaListener`)

**Lo que hicimos en Clases 5-6**:

```java
@KafkaListener(topics = "ecommerce.orders.placed")
public void consumeOrder(OrderPlacedEvent event) {
    // Lógica imperativa
    InventoryItem item = inventoryRepository.findByProductId(event.getProductId());
    item.reserveStock(event.getQuantity());
    inventoryRepository.save(item);
}
```

**Características**:
- Procesa eventos **uno por uno**
- Lógica **imperativa** (Java tradicional)
- **Stateless** (o estado en base de datos externa)
- **Caso de uso**: Actualizar base de datos, enviar notificaciones, lógica de negocio

### Kafka Streams

**Lo que haremos en Clase 7**:

```java
KStream<String, OrderConfirmedEvent> stream = builder.stream("ecommerce.orders.confirmed");
stream
    .groupBy((key, value) -> "confirmed")
    .count(Materialized.as("confirmed-count-store"));
```

**Características**:
- Procesa **streams** con transformaciones
- DSL **declarativo** (`map`, `filter`, `reduce`)
- **Stateful** (state stores locales)
- **Caso de uso**: Agregaciones, analytics, transformaciones en tiempo real

### ¿Cuándo usar cada uno?

| Usa @KafkaListener | Usa Kafka Streams |
|-------------------|-------------------|
| Actualizar base de datos | Contar eventos |
| Enviar notificaciones | Calcular promedios |
| Ejecutar lógica de negocio | Top N elementos |
| Eventos uno por uno | Agregaciones temporales |
| CRUD operations | Analytics en tiempo real |

**En nuestro curso**:
- **order-service, inventory-service**: Usan `@KafkaListener` (lógica de negocio)
- **analytics-service**: Usa Kafka Streams (agregaciones)

---

## KStream vs KTable

### KStream - Stream de Eventos (Append-Only)

**Concepto**: Flujo infinito de eventos donde **cada evento es independiente**.

**Características**:
- Representa **hechos que ocurrieron** (immutable)
- Eventos **nunca se eliminan** (append-only log)
- Cada mensaje se procesa **individualmente**

**Ejemplo**: `ecommerce.orders.placed`

```
Evento 1: orderId=1, productId=1, quantity=5
Evento 2: orderId=2, productId=1, quantity=3
Evento 3: orderId=3, productId=2, quantity=10
```

**Todos los eventos se procesan** (total: 3 órdenes creadas).

**Analogía con SQL**: `INSERT INTO orders (...) VALUES (...)`

### KTable - Vista Materializada (Changelog)

**Concepto**: Vista actualizable donde **solo importa el último valor por key**.

**Características**:
- Representa **estado actual** (mutable)
- Nuevos eventos **sobrescriben** valores anteriores para la misma key
- Solo el **último mensaje por key** importa

**Ejemplo**: `ecommerce.inventory.updated`

```
Evento 1: productId=1, availableStock=50
Evento 2: productId=1, availableStock=45  ← Sobrescribe anterior
Evento 3: productId=1, availableStock=42  ← Sobrescribe anterior
```

**Solo el último valor importa** (productId=1 → 42 disponibles).

**Analogía con SQL**: `UPDATE inventory SET availableStock=42 WHERE productId=1`

### Comparación Visual

```
KStream (ecommerce.orders.placed):
┌───────┬────────────┬──────────┐
│ Key   │ Value      │ Action   │
├───────┼────────────┼──────────┤
│ ord-1 │ Order(...)  │ Process  │
│ ord-2 │ Order(...)  │ Process  │
│ ord-3 │ Order(...)  │ Process  │
└───────┴────────────┴──────────┘
Resultado: 3 eventos procesados

KTable (ecommerce.inventory.updated):
┌───────┬────────────┬──────────┐
│ Key   │ Value      │ Action   │
├───────┼────────────┼──────────┤
│ prod-1│ Stock=50   │ Upsert   │
│ prod-1│ Stock=45   │ Upsert   │ ← Reemplaza anterior
│ prod-1│ Stock=42   │ Upsert   │ ← Reemplaza anterior
└───────┴────────────┴──────────┘
Resultado: 1 valor final (Stock=42)
```

### Log Compaction

Para que un topic funcione como KTable, debe tener **log compaction** habilitado:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create \
  --topic ecommerce.inventory.updated \
  --config cleanup.policy=compact
```

**Efecto**: Kafka **elimina mensajes antiguos** con la misma key, reteniendo solo el último.

---

## State Stores

### ¿Qué es un State Store?

Un **almacenamiento local** en cada instancia de Kafka Streams para mantener estado de forma **persistente** y **fault-tolerant**.

### Tipos de State Stores

1. **KeyValueStore** (el más común)
   - `Map<K, V>` persistente
   - Ejemplo: Contadores, agregaciones

2. **WindowStore**
   - Datos con ventanas temporales
   - Ejemplo: Agregaciones por hora

3. **SessionStore**
   - Sesiones de actividad
   - Ejemplo: Sesiones de usuario

**En este curso**: Solo usamos `KeyValueStore`.

### Características

**Fault-Tolerant**:
- Estado respaldado automáticamente en Kafka (changelog topic)
- Si una instancia falla, otra puede reconstruir el estado desde Kafka

**Local**:
- Cada instancia de la aplicación tiene su propia copia del state store
- **NO es compartido** entre instancias (pero consistente vía changelog)

**Queryable**:
- Exponer vía REST API (Interactive Queries)
- Consultar estado sin acceder a base de datos

**Persistente**:
- Por defecto usa RocksDB (almacenamiento en disco)
- Sobrevive reinicios de la aplicación

### Ejemplo de Uso

**Crear state store**:

```java
stream
    .groupByKey()
    .count(Materialized.as("my-count-store"));  // ← Crea state store
```

**Consultar state store**:

```java
ReadOnlyKeyValueStore<String, Long> store =
    kafkaStreams.store(
        StoreQueryParameters.fromNameAndType(
            "my-count-store",
            QueryableStoreTypes.keyValueStore()
        )
    );

Long count = store.get("my-key");
```

### Changelog Topic

Kafka Streams crea automáticamente un **changelog topic** para respaldar state stores:

```
State Store: "order-confirmed-count-store"
    ↓
Changelog Topic: "analytics-service-order-confirmed-count-store-changelog"
```

**Propósito**: Si la aplicación se reinicia, reconstruye state stores desde changelog topics.

---

## Operaciones Stateless vs Stateful

### Operaciones Stateless (No requieren estado)

**Características**:
- No necesitan state stores
- Rápidas, sin overhead
- Procesan eventos individualmente

**Operaciones comunes**:

| Operación | Descripción | Ejemplo |
|-----------|-------------|---------|
| `map()` | Transforma cada evento | Convertir ProductEvent a ProductDTO |
| `filter()` | Filtra eventos | Solo órdenes > $100 |
| `flatMap()` | Un evento → múltiples | Separar items de una orden |
| `selectKey()` | Cambiar key | De orderId a productId |
| `mapValues()` | Transforma solo value | Extraer quantity de OrderEvent |

**Ejemplo**:

```java
stream
    .filter((key, value) -> value.getTotalAmount() > 100)
    .mapValues(order -> order.getCustomerName());
```

### Operaciones Stateful (Requieren state stores)

**Características**:
- Requieren mantener estado
- Más lentas (acceso a disco con RocksDB)
- Fault-tolerant (respaldadas en Kafka)

**Operaciones comunes**:

| Operación | Descripción | Ejemplo |
|-----------|-------------|---------|
| `groupBy()` | Agrupar por key | Agrupar órdenes por productId |
| `count()` | Contar eventos | Contar órdenes confirmadas |
| `reduce()` | Agregaciones (sum, max, min) | Sumar cantidades vendidas |
| `aggregate()` | Agregaciones complejas | Promedio de ventas |

**Ejemplo**:

```java
stream
    .groupBy((key, value) -> "confirmed")
    .count(Materialized.as("confirmed-count-store"));  // ← Stateful
```

---

## Topología de Procesamiento

Kafka Streams organiza operaciones en un **grafo dirigido acíclico (DAG)** llamado **topología**.

### Ejemplo de Topología

```
Source (topic: ecommerce.orders.confirmed)
   ↓
filter() - Filtrar órdenes > $100
   ↓
map() - Extraer productId
   ↓
groupByKey() - Agrupar por productId
   ↓
count() - Contar por producto
   ↓
Sink (state store: product-count-store)
```

### Componentes de Topología

**Source**:
- Lee datos desde Kafka topics
- Entrada del stream

**Processor**:
- Transformaciones (map, filter, groupBy, count)
- Lógica de procesamiento

**Sink**:
- Escribe datos a Kafka topics o state stores
- Salida del stream

### Visualizar Topología

```java
Topology topology = builder.build();
System.out.println(topology.describe());
```

---

## Patrón CQRS

**CQRS (Command Query Responsibility Segregation)**: Separar modelos de escritura (commands) y lectura (queries).

### Motivación

**Problema**: Consultas complejas (analytics, reportes) impactan performance del sistema transaccional.

**Solución**: Separar modelos optimizados para cada caso de uso.

### En Nuestro Sistema

**Write Models** (Clases 5-6):

```
order-service (8081)
   ↓ Escribe órdenes
PostgreSQL ecommerce_orders
   ↓ Publica eventos
ecommerce.orders.placed
ecommerce.orders.confirmed
```

**Read Model** (Clase 7):

```
analytics-service (8083)
   ↓ Consume eventos
Kafka Streams + State Stores
   ↓ Expone via REST
GET /api/analytics/orders/stats
GET /api/analytics/products/top
```

### Ventajas

1. **Escalabilidad independiente**: Read model puede escalar sin afectar writes
2. **Modelos optimizados**: Cada modelo para su caso de uso específico
3. **Performance**: Analytics NO consulta base de datos transaccional
4. **Resiliencia**: Falla en analytics NO afecta órdenes

### Desventajas

1. **Eventual consistency**: Read model tiene retraso (segundos)
2. **Complejidad**: Mantener dos modelos sincronizados
3. **Duplicación**: Mismo dato en write y read models

**En producción**: Las ventajas superan las desventajas para sistemas de alta escala.

---

## Eventual Consistency

### Concepto

**Eventual Consistency**: El sistema **eventualmente** alcanzará un estado consistente, pero puede haber **retraso temporal**.

### En Nuestro Sistema

```
1. Usuario crea orden
   order-service → PostgreSQL (IMMEDIATELY)

2. Evento publicado a Kafka
   ecommerce.orders.placed (MILLISECONDS)

3. Analytics procesa evento
   analytics-service → State Store (MILLISECONDS)

4. Usuario consulta analytics
   GET /api/analytics/orders/stats (READ MODEL)

Total: 1-2 segundos de retraso
```

### Implicaciones

**Caso de uso válido**:
- Dashboard de analytics
- Reportes ejecutivos
- Estadísticas generales

**Caso de uso NO válido**:
- Validaciones en tiempo real
- Decisiones críticas de negocio
- Datos que deben ser 100% actualizados

**Ejemplo**:

```bash
# Usuario crea orden
curl -X POST http://localhost:8081/api/orders ...
# Respuesta: Order ID 123

# Inmediatamente consulta analytics
curl http://localhost:8083/api/analytics/orders/stats
# Respuesta: { confirmedOrders: 10 } ← SIN la orden 123 todavía

# 2 segundos después
curl http://localhost:8083/api/analytics/orders/stats
# Respuesta: { confirmedOrders: 11 } ← CON la orden 123
```

**Esto es NORMAL en arquitecturas event-driven**.

---

## Interactive Queries

### Concepto

**Interactive Queries**: Consultar state stores vía REST API sin acceder a base de datos.

### Arquitectura

```
analytics-service (Puerto 8083)
   ↓
┌──────────────────────────────┐
│ REST Controller              │
│   GET /api/analytics/...     │
└──────────┬───────────────────┘
           ↓
┌──────────────────────────────┐
│ AnalyticsQueryService        │
│   queryService.getOrderStats()│
└──────────┬───────────────────┘
           ↓
┌──────────────────────────────┐
│ Kafka Streams State Stores   │
│   order-confirmed-count-store│
│   product-sales-store        │
└──────────────────────────────┘
```

### Implementación

**Paso 1: Crear state store** (en stream):

```java
stream
    .groupBy((key, value) -> "confirmed")
    .count(Materialized.as("order-confirmed-count-store"));
```

**Paso 2: Consultar state store** (en service):

```java
ReadOnlyKeyValueStore<String, Long> store =
    kafkaStreams.store(
        StoreQueryParameters.fromNameAndType(
            "order-confirmed-count-store",
            QueryableStoreTypes.keyValueStore()
        )
    );

Long count = store.get("confirmed");
```

**Paso 3: Exponer via REST** (en controller):

```java
@GetMapping("/orders/stats")
public ResponseEntity<OrderStatsResponse> getOrderStats() {
    return ResponseEntity.ok(queryService.getOrderStats());
}
```

### Ventajas

- **Sin base de datos**: No requiere PostgreSQL, MongoDB, etc.
- **Baja latencia**: Datos en memoria (RocksDB)
- **Escalable**: State stores distribuidos entre instancias

---

## Resumen de Conceptos

| Concepto | Descripción | Uso en Clase 7 |
|----------|-------------|----------------|
| **Kafka Streams** | Biblioteca Java para procesamiento de streams | analytics-service |
| **KStream** | Stream de eventos append-only | Órdenes confirmadas/canceladas |
| **KTable** | Vista materializada (changelog) | Inventario actualizado |
| **State Stores** | Almacenamiento local fault-tolerant | Contadores, agregaciones |
| **Stateless Ops** | Transformaciones sin estado | map, filter, selectKey |
| **Stateful Ops** | Operaciones con estado | count, reduce, aggregate |
| **CQRS** | Separación write/read models | order-service vs analytics-service |
| **Eventual Consistency** | Retraso temporal en sincronización | 1-2 segundos |
| **Interactive Queries** | Consultar state stores via REST | GET /api/analytics/* |

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Flujo de eventos | Stream | Secuencia infinita de eventos |
| Almacenamiento de estado | State Store | Almacenamiento local fault-tolerant |
| Operación sin estado | Stateless Operation | Transformación sin memoria |
| Operación con estado | Stateful Operation | Transformación con memoria |
| Vista materializada | Materialized View | KTable - último valor por key |
| Topología | Topology | Grafo de procesamiento |
| Consultas interactivas | Interactive Queries | Consultar state stores via API |
| Consistencia eventual | Eventual Consistency | Sincronización con retraso |

---

**Siguiente**: [Cheatsheet - Clase 7](cheatsheet.md)
