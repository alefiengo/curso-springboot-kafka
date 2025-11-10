# Clase 6: Consumidores e Integración de Microservicios

Implementar consumidores de eventos Kafka en múltiples microservicios para lograr comunicación asíncrona bidireccional, creando el tercer microservicio (inventory-service) y completando el flujo de validación de órdenes mediante eventos.

---

## Objetivos de aprendizaje

- Crear inventory-service como tercer microservicio de la arquitectura
- Implementar consumidores con @KafkaListener en Spring Kafka
- Configurar deserialización JSON para eventos entrantes
- Entender consumer groups y offset management
- Implementar comunicación asíncrona bidireccional entre microservicios
- Validar stock de productos mediante eventos
- Actualizar estado de órdenes basándose en eventos de inventario
- Comprender eventual consistency en arquitecturas event-driven
- Ejecutar y probar flujo completo end-to-end con 3 microservicios

---

## Requisitos previos

- **Clase 5 completada**: product-service y order-service con productores funcionando
- **Docker Compose ejecutándose**: Kafka y PostgreSQL activos
  ```bash
  # Verificar servicios activos:
  cd ~/workspace/kafka-infrastructure
  docker compose ps  # Debería mostrar kafka y postgres "Up"
  ```
- **product-service corriendo** en puerto 8080 con producer de ProductCreatedEvent
- **order-service corriendo** en puerto 8081 con producer de OrderPlacedEvent
- **Maven y Java 17** configurados correctamente
- **Postman o curl** para probar endpoints REST

---

## Hoja de ruta sugerida

Antes de iniciar revisa los **[Conceptos de la clase](CONCEPTOS.md)**.

1. Presentación teórica: Consumer groups, offsets, deserialización
2. **[Lab 00: Crear inventory-service](labs/00-crear-inventory-service/)** - Tercer microservicio con CRUD de inventario
3. **[Lab 01: Consumer en inventory-service](labs/01-consumer-inventory/)** - Escuchar OrderPlacedEvent
4. **[Lab 02: Producer en inventory-service](labs/02-producer-inventory/)** - Publicar OrderConfirmed/Cancelled
5. **[Lab 03: Consumer en order-service](labs/03-consumer-order/)** - Actualizar estado de órdenes
6. **[Lab 04: Flujo completo end-to-end](labs/04-flujo-completo/)** - Probar los 3 microservicios integrados

---

## Laboratorios de la clase

- **[Lab 00: Crear inventory-service](labs/00-crear-inventory-service/README.md)** - Microservicio de gestión de inventario
- **[Lab 01: Consumer en inventory-service](labs/01-consumer-inventory/README.md)** - @KafkaListener para OrderPlacedEvent
- **[Lab 02: Producer en inventory-service](labs/02-producer-inventory/README.md)** - Publicar eventos de confirmación/cancelación
- **[Lab 03: Consumer en order-service](labs/03-consumer-order/README.md)** - Actualizar órdenes según inventario
- **[Lab 04: Flujo completo end-to-end](labs/04-flujo-completo/README.md)** - Integración y testing de arquitectura completa

---

## Arquitectura final

Al completar esta clase, tendrás 3 microservicios comunicándose asíncronamente:

```
┌─────────────────────┐      ┌─────────────────────┐      ┌─────────────────────┐
│  product-service    │      │   order-service     │      │ inventory-service   │
│  (puerto 8080)      │      │   (puerto 8081)     │      │  (puerto 8082)      │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ PostgreSQL          │      │ PostgreSQL          │      │ PostgreSQL          │
│ ecommerce           │      │ ecommerce_orders    │      │ ecommerce_inventory │
├─────────────────────┤      ├─────────────────────┤      ├─────────────────────┤
│ PRODUCER:           │      │ PRODUCER:           │      │ PRODUCER:           │
│ products.created    │      │ orders.placed       │      │ orders.confirmed    │
│                     │      │                     │      │ orders.cancelled    │
│                     │      │ CONSUMER:           │      │                     │
│                     │      │ orders.confirmed    │      │ CONSUMER:           │
│                     │      │ orders.cancelled    │      │ orders.placed       │
└─────────────────────┘      └─────────────────────┘      └─────────────────────┘
```

**Flujo de eventos**:

1. Cliente crea orden en order-service (REST POST)
2. order-service publica `OrderPlacedEvent` → Kafka
3. inventory-service CONSUME `OrderPlacedEvent`
4. inventory-service valida stock del producto
5. inventory-service publica:
   - `OrderConfirmedEvent` (si hay stock) O
   - `OrderCancelledEvent` (si no hay stock)
6. order-service CONSUME estos eventos
7. order-service actualiza estado: PENDING → CONFIRMED/CANCELLED

---

## Topics Kafka de la clase

Esta clase introduce 2 nuevos topics:

| Topic | Producer | Consumer | Propósito |
|-------|----------|----------|-----------|
| `ecommerce.orders.confirmed` | inventory-service | order-service | Orden validada, stock reservado |
| `ecommerce.orders.cancelled` | inventory-service | order-service | Orden rechazada, stock insuficiente |

**Topics existentes** (de Clase 5):
- `ecommerce.products.created` - ProductCreatedEvent
- `ecommerce.orders.placed` - OrderPlacedEvent

---

## Tarea

[Tarea para Casa](tarea/README.md)

**Objetivo**: Implementar evento `InventoryUpdatedEvent` y publicarlo cada vez que cambie el stock.

---

## Recursos

- [Cheatsheet - Clase 6](cheatsheet.md)
- [FAQ - Clase 6](FAQ.md)
- [Conceptos - Clase 6](CONCEPTOS.md)
- **[Colección Postman](recursos/postman/clase6-microservicios.postman_collection.json)** - Endpoints para probar los 3 microservicios
- [Spring Kafka Consumers](https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/listener-annotation.html)
- [Consumer Groups](https://kafka.apache.org/documentation/#consumergroups)
- [Offset Management](https://kafka.apache.org/documentation/#consumerconfigs)

---

## Próximos pasos

En la **Clase 7** agregaremos analytics-service con Kafka Streams para procesamiento en tiempo real y patrones CQRS.