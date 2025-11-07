# Lab 00: Conceptos de Apache Kafka

## Objetivo

Comprender la arquitectura fundamental de Apache Kafka, sus componentes principales (brokers, topics, partitions, producers, consumers, consumer groups) y cómo se aplican al dominio e-commerce del curso.

---

## Comandos a ejecutar

Este lab es teórico, NO requiere comandos. Los labs siguientes (01 y 02) pondrán en práctica estos conceptos con Docker Compose y Kafka CLI.

---

## Desglose del comando

No aplica para este lab teórico.

---

## Explicación detallada

### Parte 1: ¿Qué es Apache Kafka?

Apache Kafka es una **plataforma de streaming distribuida** diseñada para:

- **Publicar y suscribirse** a flujos de eventos (pub/sub)
- **Almacenar** eventos en orden y con durabilidad
- **Procesar** eventos en tiempo real

#### Diferencias con otros sistemas de mensajería

| Característica | Kafka | RabbitMQ | ActiveMQ |
|----------------|-------|----------|----------|
| **Persistencia** | Disco (logs) | Memoria/Disco | Memoria/Disco |
| **Orden garantizado** | Sí (por partición) | Sí (por cola) | Sí (por cola) |
| **Replay de mensajes** | Sí (retención configurable) | No (se eliminan al consumirse) | No |
| **Escalabilidad horizontal** | Excelente (particiones) | Buena (clustering) | Moderada |
| **Uso principal** | Event streaming, logs | Cola de tareas | Mensajería empresarial (JMS) |

**Por qué Kafka en microservicios**:

- **Desacoplamiento**: Productores y consumidores no se conocen entre sí
- **Alta disponibilidad**: Réplicas de datos entre brokers
- **Escalabilidad**: Agregar particiones y consumidores sin afectar productores
- **Auditoría**: Historia completa de eventos para troubleshooting

#### Casos de uso reales

**E-commerce** (nuestro dominio):
- Publicar eventos de órdenes → consumir en inventory-service
- Publicar eventos de productos → consumir en analytics-service
- Saga pattern para transacciones distribuidas

**Banking**:
- Detección de fraude en tiempo real
- Procesamiento de transacciones
- Event sourcing para auditoría regulatoria

**Telecommunications**:
- Procesamiento de eventos de red
- Billing y facturación
- Monitoreo de calidad de servicio (QoS)

**IoT**:
- Ingestión de telemetría de sensores
- Procesamiento de datos de vehículos conectados
- Smart cities (semáforos, cámaras, sensores ambientales)

---

### Parte 2: Componentes de Kafka

#### 1. Broker

Un **broker** es un servidor Kafka responsable de:

- Recibir mensajes de productores
- Almacenar mensajes en disco (logs)
- Servir mensajes a consumidores
- Replicar datos entre brokers

**Cluster de Kafka**:

```
┌────────────────────────────────────────┐
│           Kafka Cluster                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │Broker 1 │ │Broker 2 │ │Broker 3 │  │
│  │(Leader) │ │(Follower│ │(Follower│  │
│  │  9092   │ │  9093)  │ │  9094)  │  │
│  └─────────┘ └─────────┘ └─────────┘  │
└────────────────────────────────────────┘
```

**En este curso**: Usaremos 1 broker en desarrollo, pero en producción real se usan 3+ brokers para alta disponibilidad.

#### 2. Topic

Un **topic** es un canal lógico donde se publican y consumen mensajes. Es comparable a:

- Una **tabla** en base de datos
- Una **carpeta** en sistema de archivos
- Un **stream** de eventos relacionados

**Convención de nombres** (curso):

```
app-name.domain.event-type
```

**Ejemplos del dominio e-commerce**:

```
ecommerce.products.created
ecommerce.products.updated
ecommerce.orders.placed
ecommerce.orders.confirmed
ecommerce.orders.cancelled
ecommerce.inventory.updated
ecommerce.inventory.insufficient
```

**¿Por qué esta convención?**

- **app-name**: Identifica el sistema (evita colisiones con otros sistemas)
- **domain**: Contexto de negocio (products, orders, inventory)
- **event-type**: Acción que ocurrió (created, placed, confirmed)

**NO usar**:
- ❌ `my-topic` (genérico, sin contexto)
- ❌ `test` (no describe propósito)
- ❌ `orders` (no indica qué evento ocurrió)

#### 3. Partition

Cada topic se divide en **particiones** para lograr paralelismo y escalabilidad.

**Diagrama de particiones**:

```
Topic: ecommerce.orders.placed (3 particiones)

┌────────────────────┬────────────────────┬────────────────────┐
│   Partition 0      │   Partition 1      │   Partition 2      │
├────────────────────┼────────────────────┼────────────────────┤
│ Order 1 (offset 0) │ Order 2 (offset 0) │ Order 3 (offset 0) │
│ Order 4 (offset 1) │ Order 5 (offset 1) │ Order 6 (offset 1) │
│ Order 7 (offset 2) │ Order 8 (offset 2) │ Order 9 (offset 2) │
└────────────────────┴────────────────────┴────────────────────┘
```

**Ventajas de las particiones**:

1. **Paralelismo**: Múltiples consumidores leen particiones diferentes simultáneamente
2. **Escalabilidad**: Distribuir carga entre brokers
3. **Orden garantizado**: Mensajes dentro de una partición mantienen su orden

**Partition Key**:

El productor puede especificar una clave para determinar a qué partición va cada mensaje:

```java
// Mensajes con el mismo orderId van a la misma partición
kafkaTemplate.send("ecommerce.orders.placed", orderId, orderEvent);
//                  topic                      key        value
```

**Estrategia de particionamiento**:

- Si NO se especifica clave → Round-robin entre particiones
- Si se especifica clave → Hash(clave) % número_particiones

**¿Cuántas particiones crear?**

Regla general: `número de particiones ≥ número máximo de consumidores esperados`

- 1 partición → Solo 1 consumidor activo (no hay paralelismo)
- 3 particiones → Hasta 3 consumidores en paralelo
- 10 particiones → Hasta 10 consumidores en paralelo

**En este curso**: Usaremos 3 particiones por topic.

#### 4. Producer (Productor)

Un **producer** es una aplicación que publica mensajes a un topic.

**Responsabilidades**:

- Crear mensajes (eventos)
- Especificar topic destino
- Opcionalmente especificar partition key
- Manejar serialización (Java object → JSON/Avro/etc.)

**Diagrama de flujo**:

```
┌──────────────────┐
│  order-service   │
│                  │
│  createOrder()   │
└────────┬─────────┘
         │ 1. Crea OrderPlacedEvent
         │
         ↓
   ┌────────────────────┐
   │ OrderEventProducer │
   │  (KafkaTemplate)   │
   └─────────┬──────────┘
             │ 2. Serializa a JSON
             │
             ↓
      ┌──────────────┐
      │ Kafka Broker │
      │   Topic:     │
      │ orders.placed│
      └──────────────┘
```

**Ejemplo de código** (veremos en Clase 5):

```java
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    public void publishOrderPlaced(OrderPlacedEvent event) {
        kafkaTemplate.send("ecommerce.orders.placed", event.getOrderId(), event);
    }
}
```

**Modos de envío**:

1. **Fire-and-forget**: Envía mensaje sin esperar confirmación (rápido, pero no garantiza entrega)
2. **Sincrónico**: Espera confirmación del broker (lento, pero garantiza entrega)
3. **Asincrónico con callback**: Envía mensaje y procesa resultado después (balance)

#### 5. Consumer (Consumidor)

Un **consumer** es una aplicación que lee mensajes de un topic.

**Responsabilidades**:

- Suscribirse a topics
- Poll (solicitar) mensajes del broker
- Procesar mensajes (lógica de negocio)
- Commitear offset (marcar mensaje como procesado)

**Diagrama de flujo**:

```
      ┌──────────────┐
      │ Kafka Broker │
      │   Topic:     │
      │ orders.placed│
      └──────┬───────┘
             │ 3. Envía mensaje
             │
             ↓
   ┌────────────────────┐
   │ OrderEventConsumer │
   │  (@KafkaListener)  │
   └─────────┬──────────┘
             │ 4. Deserializa JSON
             │
             ↓
┌──────────────────┐
│ inventory-service│
│                  │
│ validateStock()  │
└──────────────────┘
```

**Ejemplo de código** (veremos en Clase 5):

```java
@Service
public class OrderEventConsumer {

    @KafkaListener(
        topics = "ecommerce.orders.placed",
        groupId = "inventory-service"
    )
    public void consumeOrderPlaced(OrderPlacedEvent event) {
        log.info("Recibido: {}", event.getOrderId());
        // Lógica de negocio
    }
}
```

**Offset**:

Cada mensaje en una partición tiene un **offset** (identificador único secuencial):

```
Partition 0 de "ecommerce.orders.placed"
┌────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ <- Offsets
└────┴────┴────┴────┴────┴────┘
          ↑
  Consumidor leyó hasta offset 2
  Próximo mensaje: offset 3
```

**Commit de offset**:

- **Auto-commit** (por defecto): Kafka commitea cada 5 segundos automáticamente
- **Manual commit**: El consumidor controla cuándo commitear

**Estrategias de lectura**:

- `auto-offset-reset: earliest` → Leer desde el primer mensaje del topic
- `auto-offset-reset: latest` → Leer solo mensajes nuevos

#### 6. Consumer Group

Un **consumer group** es un conjunto de consumidores que trabajan juntos para procesar un topic.

**Diagrama de consumer group**:

```
Topic: ecommerce.orders.placed (3 particiones)

┌────────────┬────────────┬────────────┐
│Partition 0 │Partition 1 │Partition 2 │
└─────┬──────┴─────┬──────┴─────┬──────┘
      │            │            │
      │            │            │
┌─────▼──────┬─────▼──────┬─────▼──────┐
│Consumer 1  │Consumer 2  │Consumer 3  │
│(inventory) │(inventory) │(inventory) │
└────────────┴────────────┴────────────┘

  Consumer Group ID: "inventory-service"
```

**Ventajas**:

1. **Load balancing**: Kafka distribuye particiones entre consumidores del grupo automáticamente
2. **Fault tolerance**: Si un consumidor falla, Kafka reasigna sus particiones a otros consumidores
3. **Escalabilidad**: Agregar consumidores para procesar más rápido

**Regla crítica**:

```
Número de consumidores activos ≤ Número de particiones
```

**Ejemplos**:

- 3 particiones, 2 consumidores → Consumidor 1 procesa 2 particiones, Consumidor 2 procesa 1
- 3 particiones, 3 consumidores → Cada consumidor procesa 1 partición
- 3 particiones, 5 consumidores → 3 consumidores activos, 2 inactivos (idle)

**Multiple consumer groups**:

Diferentes microservicios pueden tener diferentes consumer groups para el mismo topic:

```
Topic: ecommerce.orders.placed

       ┌─────────────────────────────┐
       │    Kafka Broker (Topic)     │
       └──────────┬──────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
    ┌────▼────┐      ┌────▼────┐
    │Consumer │      │Consumer │
    │Group:   │      │Group:   │
    │inventory│      │analytics│
    └─────────┘      └─────────┘
```

- **inventory-service** (group: `inventory-service`) → Valida stock
- **analytics-service** (group: `analytics-service`) → Genera estadísticas

Ambos reciben los mismos eventos, pero los procesan independientemente.

#### 7. KRaft (Kafka Raft)

**KRaft** es el modo moderno de consenso de Kafka que elimina la dependencia de Zookeeper.

**¿Qué hace KRaft?**

- Elección de leader de particiones
- Almacenamiento de metadata (topics, brokers, configuración)
- Detección de brokers caídos
- Gestión del cluster

**Arquitectura KRaft**:

```
┌────────────────────────────────┐
│      Kafka Cluster (KRaft)     │
│                                │
│  ┌─────────┐ ┌─────────┐      │
│  │Broker 1 │ │Broker 2 │      │
│  │+Ctrl    │ │+Ctrl    │      │
│  │ 9092    │ │ 9092    │      │
│  └─────────┘ └─────────┘      │
│                                │
│  Controller quorum interno     │
└────────────────────────────────┘
```

**Ventajas de KRaft**:

- Arquitectura más simple (un solo servicio)
- Menos recursos (no necesita Zookeeper)
- Inicio más rápido
- Mejor escalabilidad
- Production-ready desde Kafka 3.3+
- **Modo único desde Kafka 4.0+**

**IMPORTANTE**: En este curso usamos **KRaft** por ser la arquitectura moderna y oficial de Apache Kafka.

---

### Parte 3: Arquitectura event-driven en e-commerce

#### Flujo completo: Crear una orden

**Actores**:

1. **order-service** (productor)
2. **inventory-service** (consumidor y productor)
3. **Kafka** (broker intermediario)

**Secuencia de eventos**:

```
┌──────┐        ┌─────────┐        ┌──────┐       ┌────────────┐
│Client│        │ Order   │        │Kafka │       │ Inventory  │
│      │        │ Service │        │      │       │ Service    │
└───┬──┘        └────┬────┘        └───┬──┘       └──────┬─────┘
    │ POST /orders   │                 │                  │
    │───────────────>│                 │                  │
    │                │                 │                  │
    │                │ 1. Save order   │                  │
    │                │    (status=PENDING)                │
    │                │                 │                  │
    │                │ 2. Pub: OrderPlacedEvent           │
    │                │────────────────>│                  │
    │                │                 │                  │
    │ 200 OK         │                 │ 3. Sub: OrderPlacedEvent
    │<───────────────│                 │─────────────────>│
    │                │                 │                  │
    │                │                 │    4. Validate stock
    │                │                 │                  │
    │                │                 │ 5a. If stock OK: │
    │                │                 │  Pub: OrderConfirmedEvent
    │                │ 6. Sub: OrderConfirmedEvent        │
    │                │<────────────────│<─────────────────│
    │                │                 │                  │
    │                │ 7. Update order │                  │
    │                │   (status=CONFIRMED)               │
    │                │                 │                  │
```

**Eventos involucrados**:

| Evento | Productor | Consumidores | Topic |
|--------|-----------|--------------|-------|
| `OrderPlacedEvent` | order-service | inventory-service, analytics-service | ecommerce.orders.placed |
| `OrderConfirmedEvent` | inventory-service | order-service | ecommerce.orders.confirmed |
| `OrderCancelledEvent` | inventory-service | order-service | ecommerce.orders.cancelled |
| `InventoryUpdatedEvent` | inventory-service | analytics-service | ecommerce.inventory.updated |

#### Ventajas de esta arquitectura

**Desacoplamiento**:
- `order-service` NO llama directamente a `inventory-service` con HTTP
- Si `inventory-service` está caído, el evento queda en Kafka esperando

**Escalabilidad**:
- Agregar más instancias de `inventory-service` para procesar órdenes más rápido
- No afecta a `order-service`

**Auditoría**:
- Todos los eventos quedan registrados en Kafka
- Puedes reproducir el flujo completo días después

**Extensibilidad**:
- Agregar `analytics-service` para estadísticas sin modificar order-service ni inventory-service

---

### Parte 4: Event-Driven Architecture (EDA)

#### ¿Qué es EDA?

Un patrón arquitectónico donde los servicios reaccionan a **eventos** en lugar de ser invocados directamente.

**Comparación con arquitectura tradicional**:

**Arquitectura síncrona (REST directo)**:

```
┌──────────┐  HTTP GET   ┌───────────┐
│  Order   │───────────> │ Inventory │
│ Service  │<─────────── │ Service   │
└──────────┘ 200 OK/500  └───────────┘

Problemas:
- Si Inventory Service está caído, Order Service falla
- Acoplamiento fuerte (Order necesita URL de Inventory)
- Difícil de escalar (más requests → más carga)
```

**Arquitectura event-driven (Kafka)**:

```
┌──────────┐              ┌───────────┐
│  Order   │  Pub event   │  Kafka    │  Sub event  ┌───────────┐
│ Service  │─────────────>│  Broker   │────────────>│ Inventory │
└──────────┘              └───────────┘             │ Service   │
                                                     └───────────┘

Ventajas:
- Order Service NO depende de disponibilidad de Inventory
- Desacoplamiento (Order solo sabe del topic, no de Inventory)
- Fácil agregar más consumidores (analytics, notifications)
```

#### Eventos vs Comandos

| Concepto | Tiempo verbal | Ejemplo | Semántica |
|----------|---------------|---------|-----------|
| **Evento** | Pasado | `OrderPlacedEvent` | "Algo ocurrió" (notificación) |
| **Comando** | Imperativo | `PlaceOrderCommand` | "Haz esto" (solicitud) |

**En este curso usamos eventos** (past tense):

```java
// ✅ CORRECTO
public record OrderPlacedEvent(String orderId, Instant timestamp) {}
public record ProductCreatedEvent(Long productId, String name) {}

// ❌ INCORRECTO (comandos)
public record PlaceOrderCommand(String orderId) {}
public record CreateProductCommand(String name) {}
```

#### Eventual Consistency

En sistemas distribuidos con Kafka, la consistencia es **eventual**:

```
t=0s  : Cliente crea orden → order-service guarda en DB (status=PENDING)
t=10ms: order-service publica OrderPlacedEvent → Kafka
t=20ms: inventory-service recibe evento → valida stock
t=30ms: inventory-service publica OrderConfirmedEvent → Kafka
t=40ms: order-service recibe evento → actualiza status=CONFIRMED
```

Durante 40ms, el cliente ve `status=PENDING` aunque la orden será confirmada.

**Implicaciones**:

- UI debe mostrar estados intermedios ("Procesando orden...")
- No esperar respuesta síncrona inmediata
- Diseñar para tolerancia a inconsistencias temporales

#### Patrones de diseño

**1. Event Notification**:

Simple notificación de que algo ocurrió:

```java
public record ProductCreatedEvent(Long productId, String name, Instant timestamp) {}
```

**2. Event-Carried State Transfer**:

El evento incluye toda la información necesaria (evita que consumidor haga query):

```java
public record OrderPlacedEvent(
    String orderId,
    String productId,
    Integer quantity,
    BigDecimal price,
    String customerId,
    Instant timestamp
) {}
```

**3. Event Sourcing** (teórico, NO implementado):

Almacenar eventos como fuente de verdad:

```
Estado actual: Order(id=1, status=CONFIRMED, total=500)

Event Store (Kafka topic):
1. OrderPlacedEvent(id=1, total=500)
2. OrderConfirmedEvent(id=1)
3. OrderShippedEvent(id=1)

Reconstruir estado = replay de todos los eventos
```

**4. CQRS** (Command Query Responsibility Segregation):

Separar modelo de escritura del modelo de lectura:

```
Write Model:
- order-service (escribe órdenes)
- inventory-service (escribe stock)

Read Model:
- analytics-service (lee eventos, genera estadísticas)
- Dashboard consume analytics-service
```

**Implementado en Clase 7** con `analytics-service`.

---

### Parte 5: Preparación para los siguientes labs

En el **Lab 01** desplegaremos Kafka en modo KRaft con Docker Compose:

```yaml
services:
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
```

En el **Lab 02** crearemos topics usando Kafka CLI:

```bash
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic ecommerce.orders.placed \
  --partitions 3 --replication-factor 1
```

En **Clase 5** integraremos spring-kafka en los microservicios.

---

## Conceptos aprendidos

- **Apache Kafka**: Plataforma de streaming distribuida para event-driven architecture
- **Broker**: Servidor que almacena y distribuye mensajes
- **Topic**: Canal lógico de eventos (convención: app.domain.event-type)
- **Partition**: División de topic para paralelismo y escalabilidad
- **Producer**: Aplicación que publica eventos a topics
- **Consumer**: Aplicación que lee eventos de topics
- **Consumer Group**: Conjunto de consumidores que procesan un topic en paralelo
- **Offset**: Identificador único secuencial de mensajes en partición
- **KRaft**: Modo moderno de consenso de Kafka sin dependencia de Zookeeper
- **Event-Driven Architecture**: Comunicación asíncrona basada en eventos
- **Eventual Consistency**: Consistencia lograda después de un tiempo
- **Eventos vs Comandos**: Past tense (eventos) vs imperativo (comandos)

---

## Troubleshooting

No aplica para este lab teórico. El troubleshooting de Kafka será cubierto en Labs 04 y 05.

---

## Desafío adicional

### Desafío 1: Diseñar eventos para un nuevo dominio

Diseña los eventos para un sistema de **gestión de bibliotecas** con los siguientes requisitos:

- Los usuarios pueden reservar libros
- El sistema notifica disponibilidad
- Se generan multas por retraso
- Hay estadísticas de libros más solicitados

**Diseño de eventos**:

```
library.books.reserved
library.books.borrowed
library.books.returned
library.fines.generated
library.notifications.sent
```

**Microservicios**:

1. `book-service` → Gestiona catálogo
2. `reservation-service` → Gestiona reservas
3. `fine-service` → Calcula multas
4. `notification-service` → Envía notificaciones
5. `analytics-service` → Estadísticas

### Desafío 2: Calcular particiones necesarias

Tienes un topic `ecommerce.orders.placed` con las siguientes características:

- 10,000 órdenes por segundo
- Cada consumidor procesa 500 órdenes por segundo

**¿Cuántas particiones necesitas?**

```
Particiones = Throughput requerido / Throughput por consumidor
            = 10,000 / 500
            = 20 particiones
```

**¿Cuántos consumidores necesitas?**

```
Consumidores = Número de particiones
             = 20 consumidores (uno por partición)
```

### Desafío 3: Investigar Kafka Streams

Investiga cómo **Kafka Streams** permite procesar eventos en tiempo real:

- Filtrar eventos
- Transformar eventos
- Agregar (count, sum, avg)
- Ventanas de tiempo (windowing)

Recursos:
- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Confluent Kafka Streams Tutorial](https://developer.confluent.io/tutorials/)

**Veremos Kafka Streams en Clase 7** con `analytics-service`.

---

## Recursos adicionales

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka: The Definitive Guide (O'Reilly)](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Confluent Developer Guides](https://developer.confluent.io/)
- [Martin Fowler - Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Martin Fowler - Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Chris Richardson - Microservices Patterns](https://microservices.io/patterns/data/event-driven-architecture.html)
