# Conceptos: Apache Kafka - Arquitectura Event-Driven

Esta guía cubre los conceptos fundamentales de Apache Kafka para comprender la arquitectura de mensajería distribuida y event-driven.

---

## ¿Qué es Apache Kafka?

Apache Kafka es una **plataforma de streaming distribuida** diseñada para manejar flujos de datos en tiempo real con alta disponibilidad, escalabilidad y tolerancia a fallos.

### Características principales

- **Pub/Sub**: Publicadores y suscriptores desacoplados
- **Persistencia**: Mensajes almacenados en disco (no solo en memoria)
- **Escalabilidad horizontal**: Agregar brokers para manejar más carga
- **Alta disponibilidad**: Réplicas de datos entre brokers
- **Alto rendimiento**: Millones de mensajes por segundo

### Casos de uso

- **Mensajería entre microservicios**: Comunicación asíncrona y desacoplada
- **Event Sourcing**: Almacenar eventos como fuente de verdad
- **Stream Processing**: Procesamiento en tiempo real (Kafka Streams)
- **Log Aggregation**: Centralizar logs de múltiples servicios
- **Integración de sistemas**: Conectar aplicaciones heterogéneas

---

## Conceptos fundamentales de Kafka

### 1. Broker

Un **broker** es un servidor Kafka que:

- Almacena mensajes en disco
- Responde a requests de productores y consumidores
- Maneja réplicas de particiones

Un cluster de Kafka consiste en múltiples brokers para redundancia y escalabilidad.

```
Cluster de Kafka
┌─────────────────────────────────────┐
│  Broker 1   Broker 2   Broker 3     │
│  (Leader)   (Follower) (Follower)   │
└─────────────────────────────────────┘
```

### 2. Topic

Un **topic** es un canal lógico de mensajes. Es como una tabla de base de datos o carpeta donde se almacenan eventos relacionados.

**Convención de nombres**: `app-name.domain.event-type`

Ejemplos:

```
ecommerce.orders.placed
ecommerce.orders.confirmed
ecommerce.orders.cancelled
ecommerce.inventory.updated
ecommerce.products.created
```

### 3. Partition

Cada topic se divide en **particiones** para paralelizar el procesamiento.

```
Topic: ecommerce.orders.placed
┌─────────────┬─────────────┬─────────────┐
│ Partition 0 │ Partition 1 │ Partition 2 │
│ msg1, msg4  │ msg2, msg5  │ msg3, msg6  │
└─────────────┴─────────────┴─────────────┘
```

**Ventajas de las particiones**:

- **Paralelismo**: Múltiples consumidores leen diferentes particiones simultáneamente
- **Escalabilidad**: Distribuir carga entre brokers
- **Orden garantizado**: Dentro de una partición, el orden se mantiene

**Clave de particionamiento** (partition key):

- Mensajes con la misma clave van a la misma partición
- Ejemplo: `orderId` como clave para mantener orden de eventos por pedido

### 4. Producer

Un **productor** es una aplicación que publica mensajes a un topic.

```java
@Service
public class OrderEventProducer {
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    public void publishOrderPlaced(OrderPlacedEvent event) {
        kafkaTemplate.send("ecommerce.orders.placed", event.getOrderId(), event);
        //                  topic                       key               value
    }
}
```

**Características**:

- **Fire and forget**: Envía mensajes sin esperar confirmación (modo por defecto)
- **Sincrónico**: Espera confirmación del broker
- **Asíncrono con callback**: Procesa resultado cuando llega

### 5. Consumer

Un **consumidor** es una aplicación que lee mensajes de un topic.

```java
@Service
public class OrderEventConsumer {

    @KafkaListener(
        topics = "ecommerce.orders.placed",
        groupId = "inventory-service"
    )
    public void consumeOrderPlaced(OrderPlacedEvent event) {
        log.info("Procesando orden: {}", event.getOrderId());
        // Lógica de negocio
    }
}
```

**Características**:

- **Poll-based**: Consumidor solicita mensajes al broker
- **Offset tracking**: Kafka guarda la posición de lectura del consumidor
- **At-least-once delivery**: Un mensaje puede ser procesado más de una vez

### 6. Consumer Group

Un **consumer group** es un conjunto de consumidores que trabajan juntos para procesar un topic.

```
Topic: ecommerce.orders.placed (3 particiones)

Consumer Group: inventory-service
┌─────────────┬─────────────┬─────────────┐
│ Partition 0 │ Partition 1 │ Partition 2 │
└──────┬──────┴──────┬──────┴──────┬───────┘
       │             │             │
   Consumer 1    Consumer 2    Consumer 3
```

**Ventajas**:

- **Load balancing**: Kafka distribuye particiones entre consumidores del grupo
- **Fault tolerance**: Si un consumidor falla, sus particiones se reasignan
- **Escalabilidad**: Agregar consumidores para procesar más particiones

**Regla importante**: Número de consumidores ≤ número de particiones

- Si hay 3 particiones y 5 consumidores, 2 consumidores estarán inactivos
- Si hay 3 particiones y 2 consumidores, 1 consumidor procesará 2 particiones

### 7. Offset

El **offset** es un identificador único y secuencial de cada mensaje dentro de una partición.

```
Partition 0
┌────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │  <- Offsets
└────┴────┴────┴────┴────┴────┘
          ↑
    Consumidor leyó hasta offset 2
```

**Commit de offset**:

- **Auto-commit**: Kafka commitea automáticamente cada 5 segundos (por defecto)
- **Manual commit**: El consumidor controla cuándo commitear

**Estrategias de lectura**:

- `auto-offset-reset: earliest` → Leer desde el inicio del topic
- `auto-offset-reset: latest` → Leer solo mensajes nuevos

---

## Arquitectura de un sistema con Kafka

### Flujo de mensajes

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│   Producer  │─────>│ Kafka Broker │<─────│   Consumer   │
│ (order-svc) │ pub  │   (topic)    │ sub  │ (inventory)  │
└─────────────┘      └──────────────┘      └──────────────┘
```

### Sistema e-commerce completo (objetivo final del curso)

```
┌──────────────────┐
│ product-service  │──> ecommerce.products.created
└──────────────────┘

┌──────────────────┐
│  order-service   │──> ecommerce.orders.placed
└──────────────────┘
         │
         │ pub
         ↓
   ┌─────────────┐
   │ Kafka Broker│
   │  (3 topics) │
   └─────────────┘
         │
         │ sub
         ↓
┌──────────────────┐
│inventory-service │<─── ecommerce.orders.placed
│                  │──> ecommerce.inventory.updated
└──────────────────┘
         │
         │ sub
         ↓
┌──────────────────┐
│analytics-service │<─── (Kafka Streams)
└──────────────────┘
```

---

## Zookeeper vs KRaft

### Zookeeper (usado en este curso)

- **Qué hace**: Coordina el cluster de Kafka (elección de líder, metadata, configuración)
- **Despliegue**: Proceso separado que Kafka requiere
- **Estabilidad**: Tecnología madura y probada en producción

### KRaft (Kafka Raft)

- **Qué hace**: Reemplaza Zookeeper con quorum interno de Kafka
- **Estado**: Producción-ready desde Kafka 3.3 (2022)
- **Ventaja**: Simplifica arquitectura, elimina dependencia externa

**¿Por qué usamos Zookeeper en el curso?**

- Mayor estabilidad y documentación disponible
- Configuración más simple para entornos de aprendizaje
- Ampliamente usado en producción actual

---

## Event-Driven Architecture (EDA)

### ¿Qué es arquitectura event-driven?

Un patrón arquitectónico donde los servicios se comunican mediante **eventos** en lugar de llamadas síncronas.

### Eventos vs Comandos

| Tipo | Tiempo verbal | Ejemplo | Propósito |
|------|---------------|---------|-----------|
| Evento | Pasado | `OrderPlacedEvent` | Notificar que algo ocurrió |
| Comando | Imperativo | `PlaceOrderCommand` | Solicitar que algo ocurra |

En este curso usamos **eventos** (past tense).

### Ventajas de EDA

- **Desacoplamiento**: Productores no conocen a consumidores
- **Escalabilidad**: Agregar consumidores sin modificar productores
- **Resiliencia**: Si un consumidor falla, los eventos no se pierden
- **Auditoría**: Historia completa de eventos para troubleshooting

### Eventual Consistency

En sistemas distribuidos con Kafka, la consistencia es **eventual**:

1. `order-service` publica evento `orders.placed`
2. `inventory-service` lo procesa después de unos milisegundos
3. Durante ese tiempo, el stock no está actualizado

**Estrategia**: Diseñar la aplicación para tolerar inconsistencias temporales.

---

## Patrones de diseño con Kafka

### 1. Domain Events

Eventos que representan cambios significativos en el dominio de negocio.

```java
// Evento: algo que ya ocurrió
public record OrderPlacedEvent(
    String orderId,
    String productId,
    Integer quantity,
    Instant timestamp
) {}
```

### 2. Event Sourcing (teórico)

Almacenar eventos como fuente de verdad en lugar de estado actual.

```
Estado actual: Order { status: "CONFIRMED", total: 500 }

Event Store:
- OrderPlacedEvent(orderId=1, total=500)
- OrderConfirmedEvent(orderId=1)
```

**NO implementado en el curso** por complejidad, pero se explica el concepto.

### 3. CQRS (Command Query Responsibility Segregation)

Separar modelo de escritura (commands) del modelo de lectura (queries).

```
Write Model (order-service, inventory-service)
   │
   │ pub events
   ↓
Kafka Topics
   │
   │ sub events
   ↓
Read Model (analytics-service)
   │
   └─> Dashboard con estadísticas
```

**Implementado en Clase 7** con `analytics-service`.

---

## Dominio del curso: E-commerce

### Microservicios

1. **product-service**: Catálogo de productos (Clases 2-8)
2. **order-service**: Gestión de pedidos (Clases 5-8)
3. **inventory-service**: Control de stock (Clases 6-8)
4. **analytics-service**: Estadísticas en tiempo real (Clases 7-8)

### Topics de Kafka

| Topic | Productor | Consumidores | Propósito |
|-------|-----------|--------------|-----------|
| `ecommerce.products.created` | product-service | analytics-service | Notificar nuevos productos |
| `ecommerce.orders.placed` | order-service | inventory-service | Procesar pedidos |
| `ecommerce.orders.confirmed` | inventory-service | order-service | Confirmar pedido con stock disponible |
| `ecommerce.orders.cancelled` | inventory-service | order-service | Cancelar pedido sin stock |
| `ecommerce.inventory.updated` | inventory-service | analytics-service | Actualizar métricas de stock |

### Flujo completo: Crear un pedido

```
1. Cliente → POST /api/orders → order-service
2. order-service → pub OrderPlacedEvent → ecommerce.orders.placed
3. inventory-service → sub OrderPlacedEvent → valida stock
4. Si stock OK:
   inventory-service → pub OrderConfirmedEvent → ecommerce.orders.confirmed
   order-service → sub OrderConfirmedEvent → actualiza estado a CONFIRMED
5. Si stock insuficiente:
   inventory-service → pub OrderCancelledEvent → ecommerce.orders.cancelled
   order-service → sub OrderCancelledEvent → actualiza estado a CANCELLED
```

---

## Preparación para la siguiente clase

En la Clase 5 integraremos Kafka con Spring Boot:

- Configurar `spring-kafka` en `product-service`
- Crear `KafkaProducerConfig` y `KafkaConsumerConfig`
- Implementar `ProductEventProducer`
- Crear `order-service` como segundo microservicio
- Implementar `OrderEventProducer`
- Publicar eventos al crear/actualizar productos y órdenes

---

## Glosario Técnico

| Término Español | English Term | Descripción |
|-----------------|--------------|-------------|
| Broker | Broker | Servidor Kafka que almacena y distribuye mensajes |
| Tópico | Topic | Canal lógico de mensajes en Kafka |
| Partición | Partition | División de un topic para paralelismo |
| Productor | Producer | Aplicación que publica mensajes a Kafka |
| Consumidor | Consumer | Aplicación que lee mensajes de Kafka |
| Grupo de consumidores | Consumer Group | Conjunto de consumidores que procesan un topic |
| Desplazamiento | Offset | Identificador único de mensaje en partición |
| Evento | Event | Notificación de algo que ocurrió (past tense) |
| Arquitectura event-driven | Event-Driven Architecture | Patrón de comunicación basado en eventos |
| Consistencia eventual | Eventual Consistency | Datos consistentes después de un tiempo |
