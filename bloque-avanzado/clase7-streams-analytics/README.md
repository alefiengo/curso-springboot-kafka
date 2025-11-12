# Clase 7: Kafka Streams y Patrones Event-Driven

Implementación de analytics en tiempo real con Kafka Streams, state stores, y el patrón CQRS (Command Query Responsibility Segregation).

---

## Objetivos de aprendizaje

Al finalizar esta clase, serás capaz de:

- Comprender la diferencia entre consumer tradicional (`@KafkaListener`) y Kafka Streams
- Implementar procesamiento de streams con operaciones stateless y stateful
- Distinguir entre `KStream` (stream de eventos) y `KTable` (changelog)
- Utilizar state stores para mantener agregaciones en memoria
- Implementar el patrón CQRS con read models separados del write model
- Consultar state stores vía REST API usando Interactive Queries
- Implementar analytics en tiempo real sin base de datos

---

## Requisitos previos

**Clases completadas**:
- Clase 4: Introducción a Apache Kafka
- Clase 5: Productores y Consumidores
- Clase 6: Integración de Microservicios

**Microservicios operativos**:
- product-service (puerto 8080)
- order-service (puerto 8081)
- inventory-service (puerto 8082)

**Kafka topics activos**:
- `ecommerce.products.created`
- `ecommerce.orders.placed`
- `ecommerce.orders.confirmed`
- `ecommerce.orders.cancelled`

**Herramientas**:
- Java 17
- Maven 3.6+
- Docker y Docker Compose
- PostgreSQL (vía Docker)
- Apache Kafka (vía Docker)

---

## Hoja de ruta sugerida

Antes de iniciar, revisa los **[Conceptos de la clase](CONCEPTOS.md)** para familiarizarte con Kafka Streams, KStream, KTable, y state stores.

### Parte 1: Fundamentos de Kafka Streams (1:10h)

1. **Teoría**: [Conceptos de Kafka Streams](CONCEPTOS.md)
2. [Lab 00: Conceptos de Kafka Streams](labs/00-conceptos-kafka-streams/) (15 min)
   - Consumer tradicional vs Kafka Streams
   - KStream vs KTable
   - State stores
3. [Lab 01: Crear analytics-service](labs/01-crear-analytics-service/) (25 min)
   - Cuarto microservicio (puerto 8083)
   - Dependencia kafka-streams
   - Configuración básica
4. [Lab 02: Stream básico - Contar órdenes por estado](labs/02-stream-contar-ordenes/) (30 min)
   - Primer KStream
   - Operación `count()` stateful
   - Interactive Queries
5. [Lab 03: Stream con transformación - Top productos vendidos](labs/03-stream-top-productos/) (30 min)
   - Re-keying con `selectKey()`
   - Operación `reduce()`
   - Iterar state stores

**Break opcional**: 10 minutos

### Parte 2: CQRS y Validación (40 min)

6. [Lab 04: CQRS - Read model de inventario con KTable](labs/04-ktable-inventario/) (25 min)
   - KTable vs KStream
   - Log compaction
   - Patrón CQRS
7. [Lab 05: Flujo completo end-to-end con analytics](labs/05-flujo-completo/) (15 min)
   - Validación de 4 microservicios
   - Analytics en tiempo real
   - Verificación completa

---

## Laboratorios de la clase

| Lab | Título | Duración | Conceptos |
|-----|--------|----------|-----------|
| [00](labs/00-conceptos-kafka-streams/) | Conceptos de Kafka Streams | 15 min | Teoría, comparaciones |
| [01](labs/01-crear-analytics-service/) | Crear analytics-service | 25 min | Spring Initializr, configuración |
| [02](labs/02-stream-contar-ordenes/) | Stream básico - Contar órdenes | 30 min | KStream, count(), state stores |
| [03](labs/03-stream-top-productos/) | Stream - Top productos | 30 min | selectKey(), reduce(), iterators |
| [04](labs/04-ktable-inventario/) | CQRS - Read model inventario | 25 min | KTable, log compaction, CQRS |
| [05](labs/05-flujo-completo/) | Flujo completo end-to-end | 15 min | Validación, testing |

**Tiempo total**: 2.5 horas (150 minutos)

---

## Arquitectura final

Al completar esta clase, tendrás 4 microservicios comunicándose vía Kafka:

```
┌─────────────────┐
│ product-service │ (8080)
│   PostgreSQL    │
└────────┬────────┘
         │ ecommerce.products.created
         ↓
    ┌────────┐
    │ Kafka  │
    └────┬───┘
         │
         ├─→ ecommerce.orders.placed ──→ inventory-service (8082)
         │                                     │
         ├─→ ecommerce.orders.confirmed ──────┤
         │                                     │
         ├─→ ecommerce.orders.cancelled       │
         │                                     │
         └─→ ecommerce.inventory.updated      │
                                               ↓
                                    ┌──────────────────┐
                                    │ analytics-service│ (8083)
                                    │   Kafka Streams  │
                                    │   State Stores   │
                                    └──────────────────┘
```

**Bases de datos**: 3 PostgreSQL (ecommerce, ecommerce_orders, ecommerce_inventory)

**Kafka topics**: 5 topics activos

**State stores**: 4 stores en analytics-service
- order-confirmed-count-store
- order-cancelled-count-store
- product-sales-store
- inventory-read-model-store

---

## Conceptos clave

### Kafka Streams

**Biblioteca Java** para procesamiento de streams en tiempo real:

- Procesa datos directamente desde Kafka
- Sin necesidad de cluster separado (corre en la aplicación)
- Stateful processing con state stores
- Fault-tolerant y escalable

### KStream vs KTable

**KStream** - Stream de eventos (append-only):
- Cada evento es independiente
- Representa hechos que ocurrieron
- Ejemplo: `ecommerce.orders.placed`

**KTable** - Vista materializada (changelog):
- Solo importa el último valor por key
- Representa estado actual
- Ejemplo: `ecommerce.inventory.updated`

### State Stores

Almacenamiento local para mantener estado:

- **Fault-tolerant**: Respaldado en Kafka
- **Queryable**: Consultar vía REST API (Interactive Queries)
- **Local**: Cada instancia tiene su copia
- **Persistente**: Sobrevive reinicios

### CQRS (Command Query Responsibility Segregation)

Separación de modelos de escritura y lectura:

**Write Models** (Clases 5-6):
- order-service: Escribe órdenes
- inventory-service: Escribe inventario

**Read Model** (Clase 7):
- analytics-service: Lee agregaciones optimizadas

---

## Tarea

[Tarea para Casa](tarea/README.md)

**Desafío**: Implementar endpoint `/api/analytics/customers/top` que muestre top 5 clientes por cantidad de órdenes confirmadas.

**Conceptos aplicados**:
- Re-keying por customerName
- groupByKey + count
- Iteración de state store
- Ordenamiento en memoria

---

## Recursos

### Documentación de la clase

- [Cheatsheet - Clase 7](cheatsheet.md) - Comandos y patrones rápidos
- [FAQ - Clase 7](FAQ.md) - Troubleshooting común de Kafka Streams
- [Conceptos - Clase 7](CONCEPTOS.md) - Teoría detallada

### Documentación oficial

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [Spring for Apache Kafka - Streams](https://docs.spring.io/spring-kafka/reference/html/#kafka-streams)
- [Kafka Streams DSL API](https://kafka.apache.org/35/documentation/streams/developer-guide/dsl-api.html)
- [Interactive Queries](https://kafka.apache.org/35/documentation/streams/developer-guide/interactive-queries.html)

### Lecturas complementarias

- [CQRS Pattern - Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Confluent Kafka Streams Tutorial](https://developer.confluent.io/learn-kafka/kafka-streams/get-started/)

---

## Próxima clase

**Clase 8: Producción y Seguridad**

- Spring Boot Profiles (dev, prod)
- Spring Boot Actuator (health, metrics)
- Variables de entorno (12-Factor App)
- JWT Security (proteger endpoints)
- Logging estructurado
- Consolidación del proyecto final

---

**¿Preguntas o problemas?** Consulta el [FAQ](FAQ.md) o revisa el [Cheatsheet](cheatsheet.md).
