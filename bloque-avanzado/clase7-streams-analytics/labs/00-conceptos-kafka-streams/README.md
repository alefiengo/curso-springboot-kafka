# Lab 00: Conceptos de Kafka Streams

---

## 1. Objetivo

Comprender la diferencia entre consumer tradicional (`@KafkaListener`) y Kafka Streams, y entender los conceptos fundamentales: `KStream`, `KTable`, state stores, y operaciones stateless vs stateful.

---

## 2. Comandos a ejecutar

**No hay comandos en este lab** - Solo conceptos teóricos y comparaciones.

---

## 3. Desglose del comando

No aplica para este lab teórico.

---

## 4. Explicación detallada

### 4.1 Consumer Tradicional vs Kafka Streams

Hasta ahora, en las Clases 5 y 6, hemos usado **`@KafkaListener`** para procesar eventos:

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
- **Caso de uso**: Actualizar base de datos, enviar notificaciones, ejecutar lógica de negocio

**Kafka Streams** introduce un enfoque diferente:

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

**Comparación**:

| Aspecto | @KafkaListener (Clases 5-6) | Kafka Streams (Clase 7) |
|---------|------------------------------|-------------------------|
| Propósito | Procesar eventos uno por uno | Procesar streams con transformaciones |
| Operaciones | Lógica imperativa en Java | DSL declarativo (map, filter, reduce) |
| Estado | Stateless (o BD externa) | Stateful (state stores locales) |
| Caso de uso | Actualizar BD, enviar notificación | Agregaciones, analytics, transformaciones |

### 4.2 KStream vs KTable

**KStream - Stream de eventos (append-only)**:

- Cada evento es **independiente**
- Representa **hechos que ocurrieron** (immutable)
- **Ejemplo**: `ecommerce.orders.placed` - cada orden es un evento nuevo
- **Analogía con base de datos**: `INSERT` log (nunca UPDATE/DELETE)

**KTable - Vista materializada (changelog)**:

- Solo importa el **último valor por key**
- Representa **estado actual**
- **Ejemplo**: `ecommerce.inventory.updated` - solo interesa stock actual por producto
- **Analogía con base de datos**: `SELECT * FROM table` (estado actual con UPDATE/DELETE)

**Ejemplo visual**:

```
KStream (ecommerce.orders.placed):
orderId=1, productId=1, quantity=5
orderId=2, productId=1, quantity=3
orderId=3, productId=2, quantity=10
↓
Todos los eventos se procesan (3 eventos)

KTable (ecommerce.inventory.updated):
productId=1, availableStock=50
productId=1, availableStock=45  ← Sobrescribe anterior
productId=1, availableStock=42  ← Sobrescribe anterior
↓
Solo último valor por key (productId=1 → 42)
```

### 4.3 State Stores

**¿Qué es un state store?**

Un **almacenamiento local** en cada instancia de Kafka Streams para mantener estado de forma persistente y fault-tolerant.

**Tipos de state stores**:

- **KeyValueStore**: `Map<K, V>` persistente (el más común)
- **WindowStore**: Datos con ventanas temporales
- **SessionStore**: Sesiones de actividad

**Características**:

- **Fault-tolerant**: Respaldado automáticamente en Kafka (changelog topic)
- **Local**: Cada instancia tiene su copia
- **Queryable**: Exponer vía REST API (Interactive Queries)
- **Persistente**: Sobrevive reinicios (se reconstruye desde Kafka)

**Ejemplo de creación**:

```java
// Crear state store
stream
    .groupByKey()
    .count(Materialized.as("my-count-store"));  // ← Crea state store

// Consultar desde REST controller
ReadOnlyKeyValueStore<String, Long> store =
    kafkaStreams.store(
        StoreQueryParameters.fromNameAndType(
            "my-count-store",
            QueryableStoreTypes.keyValueStore()
        )
    );

Long count = store.get("my-key");
```

### 4.4 Topología de Procesamiento

Kafka Streams organiza operaciones en un **grafo dirigido acíclico (DAG)**:

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

Cada operación se ejecuta **en orden** para cada evento.

### 4.5 Operaciones Stateless vs Stateful

**Operaciones Stateless** (no requieren estado):

- `map()` - Transformar cada evento
- `filter()` - Filtrar eventos
- `flatMap()` - Transformar un evento en múltiples
- `selectKey()` - Cambiar key del mensaje
- `mapValues()` - Transformar solo el value

**Características**: Rápidas, sin overhead, sin state store.

**Operaciones Stateful** (requieren state store):

- `groupBy()` - Agrupar por key
- `count()` - Contar eventos
- `reduce()` - Agregaciones (sum, max, min)
- `aggregate()` - Agregaciones complejas

**Características**: Más lentas, fault-tolerant, crean state stores automáticamente.

---

## 5. Conceptos aprendidos

- **@KafkaListener vs Kafka Streams**: Lógica imperativa vs DSL declarativo
- **KStream**: Stream de eventos append-only (todos los eventos se procesan)
- **KTable**: Vista materializada (solo último valor por key)
- **State Stores**: Almacenamiento local, fault-tolerant, queryable
- **Interactive Queries**: Consultar state stores vía REST API
- **Operaciones Stateless**: map, filter, flatMap (no requieren estado)
- **Operaciones Stateful**: groupBy, count, reduce (requieren state store)
- **Topología**: Grafo dirigido acíclico de operaciones
- **Changelog Topic**: Topic con `cleanup.policy=compact` para KTable

---

## 6. Troubleshooting

### Problema 1: ¿Cuándo usar @KafkaListener vs Kafka Streams?

**@KafkaListener** (Clases 5-6):
- Actualizar base de datos
- Enviar notificaciones
- Ejecutar lógica de negocio
- Procesar eventos uno por uno

**Kafka Streams** (Clase 7):
- Agregaciones en tiempo real (contadores, sumas)
- Analytics (top productos, estadísticas)
- Transformaciones complejas (joins, windowing)
- CQRS read models

### Problema 2: ¿Cómo saber si usar KStream o KTable?

**KStream** si:
- Cada evento es importante (historial)
- Necesitas procesar todos los eventos
- Ejemplo: órdenes, transacciones, logs

**KTable** si:
- Solo importa el último valor por key
- Quieres estado actual
- Ejemplo: inventario, saldo de cuenta, perfil de usuario

### Problema 3: ¿Los state stores comparten estado entre instancias?

**NO**. Cada instancia tiene su propia copia local del state store.

**Sin embargo**, están respaldados en Kafka (changelog topic), por lo que:
- Si una instancia falla, otra puede reconstruir el estado desde Kafka
- Son **fault-tolerant**
- Son **consistentes** (eventual consistency)

---

## 7. Desafío adicional

**Reflexión conceptual**:

1. **Diseña un caso de uso** donde necesites KStream para un dominio (órdenes, transacciones, etc.)
2. **Diseña un caso de uso** donde necesites KTable para el mismo dominio
3. **Explica** qué pasaría si usas KStream donde deberías usar KTable (y viceversa)

**Ejemplo de solución**:

- **KStream**: `ecommerce.orders.placed` - Necesito procesar todas las órdenes para calcular total de ventas
- **KTable**: `ecommerce.inventory.updated` - Solo me importa el stock actual por producto
- **Error común**: Usar KStream para inventario → procesarías múltiples actualizaciones del mismo producto (incorrecto)

---

## 8. Recursos adicionales

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Kafka Streams DSL API](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html)
- [KStream vs KTable](https://docs.confluent.io/platform/current/streams/concepts.html#streams-concepts-kstream)
- [State Stores](https://kafka.apache.org/35/documentation/streams/developer-guide/processor-api.html#state-stores)
- [Interactive Queries](https://kafka.apache.org/35/documentation/streams/developer-guide/interactive-queries.html)
- [Confluent Kafka Streams Tutorial](https://developer.confluent.io/learn-kafka/kafka-streams/get-started/)

---

**Siguiente**: [Lab 01: Crear analytics-service](../01-crear-analytics-service/README.md)
